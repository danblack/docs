# MINUTE

_Функции даты и времени_

#### Синтаксис


```
MINUTE( datetime )
```

#### Описание
Возвращает номер минуты в часе в указанной дате `datetime`. При указании даты без времени возвращает 0.

**Типы аргументов:**
- `datetime` — `Дата | Дата со временем`


**Возвращаемый тип**: Зависит от типов аргументов

#### Примеры

```
MINUTE(#2019-01-23 15:07:47#) = 7
```


#### Поддержка источников данных

`Материализованный датасет`, `ClickHouse 1.1`, `Microsoft SQL Server 2017 (14.0)`, `MySQL 5.6`, `PostgreSQL 9.3`