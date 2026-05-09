# Статистические отчеты направления 2A
> Данные в дашбордах и таблицах представлены за период с 18 марта 2026 г. по 31 марта 2026 г.

# Структура:
```
## Заголовок_отчета
### SQL:
-

### Дашборды
-

### Отчет
-

### Экспорт данных
-
```

---

## 1. Топ 10 квартир по расходу электроэнергии за неделю
### SQL:
```sql
SELECT year_week, apartment_no, complex, building, value_sum AS electricity_kwh_week
FROM m_stats_weekly_by_apartment
WHERE metric = 'electricity_kwh_total'
ORDER BY value_sum DESC
LIMIT 10;
```

### Дашборды
-

### Отчет
Отображение 10 квартир с наибольшим потребление электроэнергии за неделю. Столбец `electricity_kwh_week` показывает сумму потребленной электроэнергии (кВт*ч) за неделю `year_week`.

### Экспорт данных
-

## 2. Динамика среднего уровня CO2 по зданиями (дни)
### SQL:
```sql
SELECT 
    date,
	complex,
    building,
    value_avg
    CASE 
        WHEN value_avg >= 2500 THEN 'severely polluted'
    		WHEN value_avg >= 1500 THEN 'very polluted'
    		WHEN value_avg >= 1000 THEN 'polluted'
    		WHEN value_avg >= 600 THEN 'moderate'
    		WHEN value_avg >= 400 THEN 'good'
    END AS status
FROM m_stats_daily_by_building
WHERE metric = 'co2_ppm'
ORDER BY date ASC;
```

### Дашборды
-

### Отчет
Отслеживание качества воздуха по среднему значению CO2 в зданиях по дням. Значение `value_avg` - среднее значение CO2 в здании за сутки. Столбец `status` показывает текущую оценку воздуха по таблице [IAQI](https://atmotube.com/blog/indoor-air-quality-index-iaqi).

### Экспорт данных
-
