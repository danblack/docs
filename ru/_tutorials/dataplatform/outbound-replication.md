# Миграция базы данных из {{ mpg-name }}

Кластер {{ mpg-name }} поддерживает [логическую репликацию](https://www.postgresql.org/docs/current/logical-replication.html). Это позволяет мигрировать базы данных встроенными средствами {{ PG }} между разными кластерами {{ PG }} версий 10 и выше. В том числе поддерживается миграция между разными версиями: например, можно перенести базы из {{ PG }} версии 11 в версию 13.

{% note info %}

Если вы используете кластеры более старых версий, то можно мигрировать базу, [создав дамп](https://www.postgresql.org/docs/current/app-pgdump.html) и затем [восстановившись](https://www.postgresql.org/docs/current/app-pgrestore.html) из него.

{% endnote %}

Этот сценарий использования описывает, как перенести базу данных из сервиса {{ mpg-name }} в другой кластер {{ PG }} с помощью логической репликации.

Чтобы мигрировать базу данных из *кластера-источника* {{ mpg-name }} в *кластер-приемник* {{ PG }}:
1. [Перенесите схему базы данных](#migrate-schema).
1. [Настройте пользователя для управления репликацией на кластере-источнике](#configure-user).
1. [Создайте публикацию на кластере-источнике](#create-publication).
1. [Создайте подписку на кластере-приемнике](#create-subscription).
1. [Отслеживайте процесс миграции](#monitor-migration) до его завершения.
1. [Закончите миграцию](#finish-migration).

Если созданные ресурсы вам больше не нужны, [удалите их](#clear-out).

## Перед началом работы {#before-you-begin}

1. Убедитесь, что все хосты кластера-источника доступны по публичному IP-адресу, чтобы кластер-приемник мог подключаться к источнику. Подробнее см. в разделе [{#T}](../../managed-postgresql/operations/cluster-create.md).
1. Установите на хосты кластера-приемника [клиентские SSL-сертификаты {{ mpg-name }}](../../managed-postgresql/operations/connect.md#get-ssl-cert). Они требуются для успешного подключения к публично доступному кластеру-источнику.
1. При необходимости настройте межсетевой экран (firewall) и [группы безопасности](../../managed-postgresql/operations/connect.md#configuring-security-groups), чтобы можно было подключаться из кластера-приемника к кластеру-источнику, а также к каждому кластеру в отдельности (например, с помощью утилиты [psql](https://www.postgresql.org/docs/current/app-psql.html)).
1. Убедитесь, что с хостов кластера-приемника можно подключиться к хостам кластера-источника.
1. Убедитесь, что можно [подключиться к кластеру-источнику](../../managed-postgresql/operations/connect.md) с помощью SSL и к кластеру-приемнику.
1. Убедитесь, что на кластере-приемнике создана пустая база данных, в которую будет проведена миграция. 
1. Убедитесь, что в кластере-приемнике существует пользователь с полными правами на доступ к этой базе.

## Перенесите схему базы данных {#migrate-schema}

Для успешной работы логической репликации необходимо, чтобы и на источнике, и на приемнике была одинаковая схема базы данных. Чтобы перенести схему базы данных:
1. Сделайте дамп схемы базы данных кластера-источника с помощью утилиты [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html):

   ```bash
   pg_dump "host=<FQDN хоста кластера-источника> port=6432 sslmode=verify-full dbname=<имя БД> user=<имя пользователя-владельца БД>" --schema-only --no-privileges --no-subscriptions --no-publications -Fd -f <директория для дампа>
   ```
   
   FQDN хоста можно получить со [списком хостов в кластере](../../managed-postgresql/operations/hosts.md#list).
   
1. При необходимости создайте пользователей с нужными правами на доступ к объектам базы данных на кластере-приемнике.

1. Восстановите схему базы данных из дампа на кластере-приемнике с помощью утилиты [pg_restore](https://www.postgresql.org/docs/current/app-pgrestore.html):

   ```bash
   pg_restore -Fd -v --single-transaction -s --no-privileges -h <FQDN хоста-мастера кластера-приемника> -U <имя пользователя-владельца БД> -p 5432 -d <имя БД> <директория с дампом>
   ```

## Настройте пользователя для управления репликацией на кластере-источнике {#configure-user}

{{ PG }} использует модель «публикация-подписка» при выполнении логической репликации: кластер-приемник *подписывается* на *публикацию* кластера-источника, чтобы перенести данные. Чтобы успешно подписаться на публикацию, к кластеру-источнику {{ mpg-name }} нужно обращаться от имени пользователя, которому назначена роль для управления логической репликацией. Чтобы настроить такого пользователя:
1. [Создайте пользователя](../../managed-postgresql/operations/cluster-users.md#adduser).
1. [Назначьте роль](../../managed-postgresql/operations/grant.md#grant-role) `mdb_replication` этому пользователю.
1. [Подключитесь к базе данных](../../managed-postgresql/operations/connect.md), которую нужно мигрировать, от имени **владельца базы**.
1. [Выдайте созданному пользователю привилегию](../../managed-postgresql/operations/grant.md#grant-privilege) на выполнение операции `SELECT` над всеми таблицами базы данных.

После [создания подписки](#create-subscription) подключение к кластеру-источнику со стороны приемника будет осуществляться от имени этого пользователя.

## Создайте публикацию на кластере-источнике {#create-publication}

1. [Подключитесь](../../managed-postgresql/operations/connect.md) к **хосту-мастеру** и к базе данных, которую нужно мигрировать, от имени **владельца базы**.
1. Создайте публикацию, на которую будет подписываться кластер-приемник:

   ```sql
   CREATE PUBLICATION <имя публикации>;
   ```
   
1.  Включите в созданную публикацию все таблицы базы данных:

    ```
    ALTER PUBLICATION <имя публикации> ADD TABLE <имя таблицы 1>;
    ...
    ALTER PUBLICATION <имя публикации> ADD TABLE <имя таблицы N>;
    ``` 
    
    {% note info %}
    
    В кластере {{ mpg-name }} запрещено создание публикации сразу для всех таблиц `CREATE PUBLICATION ... FOR ALL TABLES;`, т.к. это требует привилегий суперпользователя.
    
    {% endnote %}
    
## Создайте подписку на кластере-приемнике {#create-subscription}

1. Подключитесь к **хосту-мастеру** и базе данных, в которую нужно выполнить миграцию, от **имени суперпользователя** (например, `postgres`).
1. Создайте подписку на публикацию кластера-источника:

   ```sql
   CREATE SUBSCRIPTION <имя подписки> CONNECTION 'host=<FQDN хоста кластера-источника> port=6432 sslmode=verify-full dbname=<имя БД, которую нужно мигрировать> user=<имя пользователя для управления репликацией> password=<пароль пользователя>' PUBLICATION <имя публикации>;
   ```
   
Начнется процесс миграции данных из базы кластера-источника в базу кластера-приемника.

## Отслеживание процесса миграции {#monitor-migration}

Следите за миграцией с помощью каталога [pg_subscription_rel](https://www.postgresql.org/docs/current/catalog-pg-subscription-rel.html), который показывает *статус репликации*:

```sql
SELECT * FROM pg_subscription_rel;

 srsubid | srrelid | srsubstate | srsublsn
---------+---------+------------+----------
...
```

Рекомендуется следить за статусом репликации по полю `srsubstate` этого каталога на кластере-приемнике.

Общий статус репликации на кластере-приемнике можно получить с помощью представления (view) [pg_stat_subscription](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-SUBSCRIPTION), на источнике — с помощью представления [pg_stat_replication](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW).

## Закончите миграцию {#finish-migration}

После завершения процесса репликации:
1. Запретите запись в смигрированную базу данных на кластере-источнике.

1. Перенесите последовательности (sequences), если они есть, из кластера-источника на кластер-приемник с помощью утилит pg_dump и psql:

   ```bash
   pg_dump "host=<FQDN хоста-мастера кластера-источника> port=6432 sslmode=verify-full dbname=<имя БД> user=<имя пользователя-владельца БД>" --data-only -t '*.*_seq' > <имя файла с sequences>
   ```
   
   ```bash
   psql -h <FQDN хоста-мастера кластера-приемника> -U <имя пользователя-владельца БД> -p 5432 -d <имя БД> < <имя файла с sequences>
   ``` 

1. Удалите подписку на кластере-приемнике:

   ```sql
   DROP SUBSCRIPTION <имя подписки>;
   ```
   
1. Удалите публикацию на кластере-источнике:

   ```sql
   DROP PUBLICATION <имя публикации>;
   ```
   
1. [Удалите пользователя](../../managed-postgresql/operations/cluster-users.md#removeuser) для управления репликацией на кластере-источнике.

## Удалите созданные ресурсы {#clear-out}

Если созданные ресурсы вам больше не нужны, удалите их:

1. [Удалите виртуальную машину](../../compute/operations/vm-control/vm-delete.md).
1. Если вы зарезервировали для виртуальной машины публичный статический IP-адрес, [удалите его](../../vpc/operations/address-delete.md).
1. [Удалите кластер {{ mpg-full-name }}](../../managed-postgresql/operations/cluster-delete.md).