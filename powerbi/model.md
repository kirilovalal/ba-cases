# Модель данных

Схема «звезда» с одним измерением. Все факты соединяются с `dim_case` по `case_id`; между собой факты не связаны.

```
                 ┌──────────────────┐
                 │     dim_case     │
                 │ PK case_id       │
                 │ company          │
                 │ case_name        │
                 │ domain           │
                 │ period_before    │
                 │ period_after     │
                 │ unit_of_analysis │
                 │ observation_win. │
                 └───┬────┬────┬────┘
          1:n       │    │    │        1:n
   ┌────────────────┘    │    └──────────────────┐
   ▼                     ▼                       ▼
fact_metric        fact_reason           fact_timing_rows
case_id (FK)       case_id (FK)          case_id (FK)
metric_id          period                period
metric_name        reason                observation_no
period             count                 active_seconds
numerator
denominator            fact_timing_agg  (case_id FK, period, n_observations, median_seconds, note)
good_direction
segment
```

## Таблицы и типы

| Таблица | Гранулярность | Ключевые поля | Типы |
|---|---|---|---|
| dim_case | одна строка на кейс | case_id (текст, уникальный) | все текст |
| fact_metric | метрика × период × сегмент | case_id, metric_id, period, segment | numerator, denominator — целое |
| fact_reason | кейс × период × причина | case_id, period, reason | count — целое |
| fact_timing_rows | одно наблюдение | case_id, period, observation_no | active_seconds — целое |
| fact_timing_agg | кейс × период | case_id, period | n_observations, median_seconds — целое |

Связи: `dim_case[case_id]` 1 → * к каждому факту, направление фильтра — одностороннее от измерения к фактам. Кросс-фильтрацию между фактами не включать: у них разная единица учёта (задача, инцидент, обращение, SKU, наблюдение), и общий срез по ним не имеет смысла.

## Что считать можно и что нельзя

Можно: долю по любому кейсу и периоду (`Share`), разницу в п.п. (`Delta pp`), распределение причин внутри периода, медиану активного времени TransitDesk из строк (`Median Seconds Rows`) и её сходимость с агрегатом.

Нельзя, и меры этого не делают: складывать числители разных кейсов (разные единицы), усреднять доли между кейсами, считать медиану KioskCare заново (исходных длительностей нет — только агрегат), выводить деньги.

## Контрольные точки после загрузки

`Reason Check` по TransitDesk, BeautyCatalog и PartnerFlow должен показывать «ок» для обоих периодов — сумма причин равна числителю метрики «Всего». `Median Seconds Before Rows` = 1044, `Median Seconds After Rows` = 738 — совпадают с `fact_timing_agg`. `Cases Count` = 6.
