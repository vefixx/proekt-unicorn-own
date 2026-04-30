# Сущность решения задачи
```
## Название_задачи
- **Описание решения:**
- **SQL-запросы, скрипты:**
- **Источники, скриншоты:**
```

# Направление 1А
## Потребление электроэнергии и воды (агрегация за час)
- **Описание решения:** так как показания в базе предоставлены за каждый час, то будем использовать оконную функцию LAG, позволяющая получать предыдущую строку. Будем получать предыдущую строку по столбцу `device_id` и `metric_type.code`, записывать предыдущее значение в столбец `prev_value` и вычитать из текущего.
- **SQL-запросы, скрипты:**
```sql
DROP VIEW IF EXISTS view_resource_consumption_hourly;
CREATE VIEW view_resource_consumption_hourly AS
WITH base_cte AS (
	SELECT
		m.measurement_id,
		m.ts,
		b.building_id,
		c.name AS complex,
		b.name AS building,
		d.apartment_id,
		a.apartment_no,
		mt.code,
		mt.unit,
		m.value_num,
		LAG(m.value_num) OVER (PARTITION BY d.device_id, mt.code ORDER BY m.ts) AS prev_value
	FROM measurement m
	JOIN device d ON d.device_id = m.device_id
	JOIN building b ON b.building_id = d.building_id
	JOIN complex c ON c.complex_id = b.complex_id
	JOIN apartment a ON a.apartment_id = d.apartment_id
	JOIN metric_type mt ON mt.metric_type_id = m.metric_type_id
	WHERE mt.code IN ('water_cold_m3_total', 'water_hot_m3_total', 'electricity_kwh_total')
)
SELECT
	measurement_id,
	ts,
	building_id,
	complex,
	building,
	apartment_id,
	apartment_no,
	unit,
	code,
	CAST(ROUND(
		CASE
			WHEN prev_value IS NULL THEN 0
			ELSE value_num - prev_value
		END,
		3
	) AS REAL) AS value
FROM base_cte;
```
- **Источники, скриншоты:** -

## Потребление электроэнергии и воды (агрегация за день)
- **Описание решения:** Для получения данных за день будем использовать уже существующее представление view_resource_consumption_hourly. Будем группировать строки по дате (функция DATE() возвращает год, месяц и день) и по всем остальным столбцам, и суммировать данные из каждой группы.
- **SQL-запросы, скрипты:**
```sql
DROP VIEW IF EXISTS view_resource_consumption_daily;

CREATE VIEW view_resource_consumption_daily AS
SELECT
    DATE(ts) AS date,
    building_id,
    building,
    complex,
    apartment_id,
    apartment_no,
    code,
    unit,
    ROUND(SUM(value), 3) AS value
FROM view_resource_consumption_hourly
GROUP BY
    DATE(ts),
    complex,
    building_id,
    building,
    apartment_id,
    apartment_no,
    code,
    unit;
```
- **Источники, скриншоты:** -

## Выявление опасных сценариев энергопотребления при отсутствии движения
- **Описание решения:** Используя представление `view_resource_consumption_hourly`, с данными о потреблении за прошедший час, посчитаем процент повышения/понижения потребления. Для каждой строки найдем разницу в процентах между текущим и предыдущим значением (sql-запрос №1).
Далее создадим представление с наличием движения в квартире (sql-запрос №2). *В перспективе лучше сгруппировать так, что если хотя бы в одной из комнат есть движение, то пометим что во всей квартире есть движение. Но посмотрев данные об устройствах этого не требуется, так как на одну квартиру установлено одно устройство*.
В конце создадим финальное представление (sql-запрос №3). Опасным сценарием будем помечать строки, где потребление выросло на +30 или более процентов и движения в это время не было.

- **SQL-запросы, скрипты:**
**SQL-запрос создания представления с процентами потребления (№1)**
```sql
DROP VIEW IF EXISTS view_energy_delta_percent_by_apartment;
CREATE VIEW view_energy_delta_percent_by_apartment AS
WITH delta_cte AS (
    SELECT
        ts,
        complex,
        building_id,
        building,
        apartment_id,
        apartment_no,
        value,
        LAG(value) OVER (PARTITION BY complex, building, apartment_id, apartment_no ORDER BY ts) AS prev_value
    FROM view_resource_consumption_hourly
    WHERE code = 'electricity_kwh_total'
)
SELECT 
    ts,
    complex,
    building_id,
    building,
    apartment_id,
    apartment_no,
    CASE 
        WHEN prev_value IS NULL OR prev_value = 0 THEN 0
        ELSE ROUND(((value - prev_value) * 100.0 / prev_value), 2)
    END AS delta_percent
FROM delta_cte
```


**Представление с наличием движения в квартире (№2)**
```sql
DROP VIEW IF EXISTS view_motion_detect_by_apartment;
CREATE VIEW view_motion_detect_by_apartment AS
WITH base_cte AS (
    SELECT
        m.ts,
        a.building_id,
        d.apartment_id,
        a.apartment_no,
        m.value_bool AS has_motion
    FROM measurement m
    JOIN device d ON d.device_id = m.device_id
    JOIN metric_type mt ON mt.metric_type_id = m.metric_type_id
    JOIN apartment a ON a.apartment_id = d.apartment_id
    WHERE mt.code = 'motion_detected'
)
SELECT
    ts,
    building_id,
    apartment_id,
    apartment_no,
    has_motion
FROM base_cte;
```

**Финальное представление (№3)**
```sql
DROP VIEW IF EXISTS view_danger_energy_with_no_motion_scen;
CREATE VIEW view_danger_energy_with_no_motion_scen AS
SELECT 
    venergy.ts,
    venergy.building_id,
    venergy.building,
    venergy.complex,
    venergy.apartment_id,
    venergy.apartment_no,
    CAST(venergy.delta_percent AS REAL) AS delta_percent,
    vmotion.has_motion,
    CASE
        WHEN venergy.delta_percent > 30 AND vmotion.has_motion == 0 THEN 1
        ELSE 0
    END AS is_danger
FROM view_energy_delta_percent_by_apartment venergy
JOIN view_motion_detect_by_apartment vmotion ON vmotion.ts = venergy.ts AND vmotion.apartment_id = venergy.apartment_id AND vmotion.building_id = venergy.building_id
```
- **Источники, скриншоты:** -

## Формирование журнала инцидентов
- **Описание решения:**
Для формирования журнала создадим физическую таблицу, в которую будем записывать все **новые** инциденты из представлений. Заполнять таблицу будем скриптом. Скрипт реализуем на языке C#. Сценарий примерно следующий:
1. Открываем базу данных
2. Получаем дату и время последнего инцидента
3. Проходимся поочередно по всем представлениям с опасными сценариями, аномалиями и т.д., получаем в них все записи, дата которых старше последнего инцидента в физ. таблице
4. Добавляем эти данные в физ. таблицу, разбивая их на категории
5. Закрываем базу
Далее будем запускать этот скрипт по расписанию. Лично я использовал свой личный VDS, добавлял запись в crontab для запуска скрипта раз в час.
*Уточнение - база данных должна находится в том же хранилище, где и скрипт (т.е. находиться на одном сервере).*

Структура физической таблицы будет следующая:
```
- incident_id (integer - primary key)
- ts (дата и время)
- building_id (ключ к таблице building)
- building (string)
- complex (string)
- apartment_id (ключ к таблице apartment)
- apartment_no (string)
- category (string)
- message (string)
```


- **SQL-запросы, скрипты:** https://github.com/vefixx/UnicornDatabaseIncidentJournalScript/tree/master
- **Источники, скриншоты:**

# Направление 1Б
## Качество воздуха
- **Описание решения:** К определению качества воздуха будем относить измерение VOC индекса, CO2 и pm2.5.
Создадим представление (sql-запрос №1), которое сопоставляет все столбцы, относящиеся к качеству воздуха в помещении. Составим так называемую сводную таблицу, где строки будут столбцами. К базовым столбцам (ts, building_id, complex_id, apartment_id, apartment_no) добавим столбцы voc_index, co2_ppm, pm25_ug_m3 с их соответствующими значениями по квартире и дате.
После этого создадим витрину данных (sql-запрос №2), где будем вычислять IAQI (Indoor Air Quality Index) для каждого значения. В качестве результата возьмем минимальный индекс и сверим с таблицей. В конце напишем запрос для обновления (sql-запрос №3)
В столбце `status` витрины поставим одно из значений:
```
- good (значение 81-100)
- moderate (значение 61-80)
- polluted (значение 41-60)
- very polluted (значение 21-40)
- severely polluted (значение 0-20)
```


- **SQL-запросы, скрипты:**
**Представление качества воздуха (№1)**
```sql
DROP VIEW IF EXISTS view_air_quality_by_apartment;
CREATE VIEW view_air_quality_by_apartment AS
SELECT
    m.ts,
    d.building_id,
    d.apartment_id,
    a.apartment_no,
    MAX(m.value_num) FILTER (WHERE mt.code = 'co2_ppm') AS co2_ppm,
    MAX(m.value_num) FILTER (WHERE mt.code = 'voc_index') AS voc_index,
    MAX(m.value_num) FILTER (WHERE mt.code = 'pm25_ug_m3') AS pm25_ug_m3
FROM measurement m
JOIN metric_type mt ON m.metric_type_id = mt.metric_type_id
JOIN device d ON m.device_id = d.device_id
JOIN apartment a ON d.apartment_id = a.apartment_id
GROUP BY m.ts, d.building_id, d.apartment_id, a.apartment_no
```

**Создание витрины (№2)**
```sql
CREATE TABLE IF NOT EXISTS m_air_quality_status_by_apartment (
    ts TEXT,
    building_id INTEGER,
    complex_id INTEGER,
    apartment_id INTEGER,
    apartment_no TEXT,
    co2_iaqi REAL,
    voc_iaqi REAL,
    pm25_iaqi REAL,
    min_iaqi REAL,
    status TEXT,
    FOREIGN KEY(building_id) REFERENCES building(building_id),
    FOREIGN KEY(complex_id) REFERENCES complex(complex_id),
    FOREIGN KEY(apartment_id) REFERENCES apartment(apartment_id)
)
```

**Заполнение/обновление витрины (№3)**
```sql
DELETE FROM m_air_quality_status_by_apartment;

WITH iaqi_cte AS (
    SELECT
        vair.ts,
        vair.building_id,
        vair.complex_id,
        vair.apartment_id,
        vair.apartment_no,
        ROUND(
            CASE
                WHEN vair.co2_ppm <= 599 THEN 100.0 - ((vair.co2_ppm - 400.0) / 199.0) * 19.0
                WHEN vair.co2_ppm <= 999 THEN 80.0 - ((vair.co2_ppm - 600.0) / 399.0) * 19.0
                WHEN vair.co2_ppm <= 1499 THEN 60.0 - ((vair.co2_ppm - 1000.0) / 499.0) * 19.0
                WHEN vair.co2_ppm <= 2499 THEN 40.0 - ((vair.co2_ppm - 1500.0) / 999.0) * 19.0
                WHEN vair.co2_ppm <= 4000 THEN 20.0 - ((vair.co2_ppm - 2500.0) / 1500.0) * 20.0
                ELSE 0
            END, 3
        ) AS co2_iaqi,
        ROUND(
            CASE
                WHEN vair.voc_index <= 199 THEN 100.0 - ((vair.voc_index - 1.0) / 198.0) * 19.0
                WHEN vair.voc_index <= 249 THEN 80.0 - ((vair.voc_index - 200.0) / 49.0) * 19.0
                WHEN vair.voc_index <= 349 THEN 60.0 - ((vair.voc_index - 250.0) / 99.0) * 19.0
                WHEN vair.voc_index <= 399 THEN 40.0 - ((vair.voc_index - 350.0) / 49.0) * 19.0
                WHEN vair.voc_index <= 500 THEN 20.0 - ((vair.voc_index - 400.0) / 100.0) * 20.0
                ELSE 0
            END, 3
        ) AS voc_iaqi,
        ROUND(
            CASE
                WHEN vair.pm25_ug_m3 <= 20 THEN 100.0 - (vair.pm25_ug_m3 / 20.0) * 19.0
                WHEN vair.pm25_ug_m3 <= 50 THEN 80.0 - ((vair.pm25_ug_m3 - 21.0) / 29.0) * 19.0
                WHEN vair.pm25_ug_m3 <= 90 THEN 60.0 - ((vair.pm25_ug_m3 - 51.0) / 39.0) * 19.0
                WHEN vair.pm25_ug_m3 <= 140 THEN 40.0 - ((vair.pm25_ug_m3 - 91.0) / 49.0) * 19.0
                WHEN vair.pm25_ug_m3 <= 200 THEN 20.0 - ((vair.pm25_ug_m3 - 141.0) / 59.0) * 20.0
                ELSE 0
            END, 3
        ) AS pm25_iaqi
    FROM view_air_quality_by_apartment vair
),
iaqi_min_result AS (
    SELECT 
        ts,
		building_id,
		complex_id,
		apartment_id,
		apartment_no,
        co2_iaqi,
		voc_iaqi,
		pm25_iaqi,
        MIN(co2_iaqi, voc_iaqi, pm25_iaqi) AS min_iaqi
    FROM iaqi_cte
)
INSERT INTO m_air_quality_status_by_apartment (
    ts,
	building_id,
	complex_id,
	apartment_id,
	apartment_no,
    co2_iaqi,
	voc_iaqi,
	pm25_iaqi,
	min_iaqi,
	status
)
SELECT 
    ts,
	building_id,
	complex_id,
	apartment_id,
	apartment_no,
    co2_iaqi,
	voc_iaqi,
	pm25_iaqi,
	min_iaqi,
    CASE 
        WHEN min_iaqi >= 81 THEN 'good'
        WHEN min_iaqi >= 61 THEN 'moderate'
        WHEN min_iaqi >= 41 THEN 'polluted'
        WHEN min_iaqi >= 21 THEN 'very polluted'
        ELSE 'severely polluted'
    END AS status
FROM iaqi_min_result;
```
- **Источники, скриншоты:** https://atmotube.com/blog/indoor-air-quality-index-iaqi

**Таблица расчета индекса**
<img width="823" height="588" alt="image" src="https://github.com/user-attachments/assets/ef118d2f-1cdd-43ce-b887-c3a95be7aaf7" />

**Формула расчета индекса среди множества метрик**
<img width="823" height="364" alt="image" src="https://github.com/user-attachments/assets/2681d6c4-6da9-49ac-b348-e17f8ce50a7e" />


## Комфорт помещения
- **Описание решения:** Ключевыми метриками для расчета комфорта в помещении будем использовать температуру и влажность - они необходимы для вычисления индекса THI. Составим представление (sql-запрос №1), где для каждой квартире сопоставим температуру и влажность в конкретном часу (по аналогии с качеством воздуха).
Далее создадим витрину данных, где перечислим все столбцы из представления, но добавим к ним еще 2 столбца: `thi_index` и `status` (sql-запрос №2). Расчитаем индекс THI (Temperature Humidity Index) по следующей формуле: `0.8 * T + RH / 100 * (room_temp_c - 14.4) + 46.4`
Где:
- T - температура воздуха
- RH - относительная влажность, представленная как десятичная дробь (процент влажности делится на 100%, например для 40% будет 0.4)
Столбец `status` заполним по формуле:
```
- comfort zone (THI < 68)
- mild stress (THI 68-71)
- moderate stress (THI 72-79)
- severe stress (THI 80-89)
- danger zone (THI > 90)
```

- **SQL-запросы, скрипты:**
**Создание представления сопоставление температуры и влажности (№1)**
```sql
DROP VIEW IF EXISTS view_comfort_by_apartment;
CREATE VIEW view_comfort_by_apartment AS
SELECT
    m.ts,
    d.building_id,
    b.complex_id,
    d.apartment_id,
    a.apartment_no,
    MAX(m.value_num) FILTER (WHERE mt.code = 'humidity_pct') AS humidity_pct,
    MAX(m.value_num) FILTER (WHERE mt.code = 'room_temp_c') AS room_temp_c
FROM measurement m
JOIN metric_type mt ON m.metric_type_id = mt.metric_type_id
JOIN device d ON m.device_id = d.device_id
JOIN apartment a ON d.apartment_id = a.apartment_id
JOIN building b ON d.building_id = b.building_id
GROUP BY m.ts, d.building_id, d.apartment_id, a.apartment_no
```

**Создание витрины (№2)**
```sql
CREATE TABLE m_comfort_status_by_apartment (
    ts TEXT,
    building_id INTEGER,
    complex_id INTEGER,
    apartment_id INTEGER,
    apartment_no TEXT,
    room_temp_c REAL,
    humidity_pct REAL,
    thi_index REAL,
    status TEXT,
    FOREIGN KEY(building_id) REFERENCES building(building_id),
    FOREIGN KEY(complex_id) REFERENCES complex(complex_id),
    FOREIGN KEY(apartment_id) REFERENCES apartment(apartment_id)
)
```

**Заполнение витрины**
```sql
DELETE FROM m_comfort_status_by_apartment;

WITH result_cte AS (
    SELECT
        ts,
        building_id,
        complex_id,
        apartment_id,
        apartment_no,
        room_temp_c,
        humidity_pct,
        ROUND(
            0.8 * room_temp_c + humidity_pct / 100 * (room_temp_c - 14.4) + 46.4,
            3
        ) AS thi_index
    FROM view_comfort_by_apartment
)
INSERT INTO m_comfort_status_by_apartment (
    ts,
    building_id,
    complex_id,
    apartment_id,
    apartment_no,
    room_temp_c,
    humidity_pct,
    thi_index,
    status
)
SELECT
    ts,
    building_id,
    complex_id,
    apartment_id,
    apartment_no,
    room_temp_c,
    humidity_pct,
    thi_index,
    CASE
        WHEN thi_index > 90 THEN 'danger zone'
        WHEN thi_index >= 80 THEN 'severe stress'
        WHEN thi_index >= 72 THEN 'moderate stress'
        WHEN thi_index >= 68 THEN 'mild stress'
        ELSE 'comform zone'
    END AS status
FROM result_cte
```
- **Источники, скриншоты:** https://www.pericoli.com/en/temperature-humidity-index-what-you-need-to-know-about-it/
<img width="908" height="233" alt="image" src="https://github.com/user-attachments/assets/e1a42b43-f6ff-4b81-a21e-402cd745e11f" />

## Эффективность автоматизации вентиляции
- **Описание решения:** построим промежуточное представление для сопоставления значения CO2 и вкл/выкл вентиляции (sql-запрос №1). Для расчета "эффективности" работы вентиляции в каждой квартире будем делать следующее:
1. Отсортируем строки по возрастанию `apartment_id`
2. Найдем строки, где положение `vent_on` переходит 0 -> 1 (в результате у нас должно быть две строки: первая с `vent_on = 0` (строка `A`), а вторая с `vent_on = 1` - она нам не нужна.
3. Начиная со строки `A` будем идти вперед, пока не дойдем до `vent_on = 0` (то есть где `vent_on` переходит 1 -> 0) или изменения `apartment_id`. Конечную строку с `vent_on = 1` назовем строкой `B`
4. В промежутке от `A` до `B` (не включая `A`) найдем минимум значения `co2_ppm` - назовем `co2_ppm_min`
5. Рассчитаем "эффективность" по формуле `(A.co2_ppm - co2_ppm_min) / A.co2_ppm * 100` = `efficiency_pct` (в %)
6. Добавим в витрину новую строку, где в столбце `vent_on_ts` будет записана дата и время включения вентиляции (`A.ts`), а в `vent_off_ts` будет записана дата и время выключения (`B.ts`)
7. И так продолжаем поиск следующего промежутка, начиная от строки `B`

В результате мы получим, что если `efficiency_pct` > 0, то вентиляция сработала хорошо на `efficiency_pct` процентов. Если же `efficiency_pct` < 0, то вентиляция наоборот - сработала хуже на `efficiency_pct` процентов.
Скриншот №1 показывает как обозначаются строки `A` и `B` при работе алгоритма для конкретной квартиры.
- **SQL-запросы, скрипты:**
Скрипт обновления витрины: https://github.com/vefixx/UnicornVentAutomationEfficiency

**Структура витрины:**
```sql
CREATE TABLE IF NOT EXISTS m_vent_automation_efficiency (
	vent_on_ts TEXT NOT NULL,
	vent_off_ts TEXT NOT NULL,
	building_id INTEGER NOT NULL,
	complex_id TEXT NOT NULL,
	apartment_id INTEGER NOT NULL,
	apartment_no TEXT NOT NULL,
	co2_ppm_min REAL NOT NULL,
	efficiency_pct REAL NOT NULL,
	duration_hours INTEGER NOT NULL,
	FOREIGN KEY (building_id) REFERENCES building(building_id),
	FOREIGN KEY (apartment_id) REFERENCES apartment(apartment_id)
)
```

**Промежуточное представление (№1)**
```sql
DROP VIEW IF EXISTS view_vent_automation_efficiency;
CREATE VIEW view_vent_automation_efficiency AS
    SELECT
        m.ts,
        b.complex_id,
        d.building_id,
        d.apartment_id,
        a.apartment_no,
        MAX(m.value_num) FILTER ( WHERE mt.code = 'co2_ppm' ) AS co2_ppm,
        MAX(m.value_bool) FILTER ( WHERE mt.code = 'ventilation_on' ) AS vent_on
    FROM measurement m
JOIN device d ON d.device_id = m.device_id
JOIN building b ON b.building_id = d.building_id
JOIN apartment a ON a.apartment_id = d.apartment_id
JOIN metric_type mt ON mt.metric_type_id = m.metric_type_id
GROUP BY m.ts, b.complex_id, d.building_id, d.apartment_id, a.apartment_no
```
- **Источники, скриншоты:**

**Пример строк работы вентиляции по алгоритму (№1)**
<img width="1105" height="281" alt="image" src="https://github.com/user-attachments/assets/052f2253-5537-48ed-a093-1791d483b621" />
