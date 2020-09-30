---
title: sql
description: шпаргалка по sql
---

## Работа с последовательностями

```sql
-- выбор последнего элемента
select last_value from schema.sequence

-- сброс последовательности
alter sequence schema."sequence" restart with 5;

```

