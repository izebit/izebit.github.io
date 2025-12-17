---
layout: post
tags: 
  - liquibase
title: Migration from mysql to postgresql with liquibase
---

Recently, I had problem about migration tables and data from one relation database to another one. 
I solved it with one remarkable tool called <a href="https://www.liquibase.org/" rel="nofollow">_liquibase_</a>.
It is cross-platform and written in java.

MySQL database is given, and we want to transfer data from it to Postgresql. 
Initially, we have to create a change log file for existing tables:

```shell
sh liquibase.sh --driver=com.mysql.jdbc.Driver \
      --classpath=<path to jdbc driver> \
      --changeLogFile=<path to some directory>/changelogs.xml \
      --url="<database url, example jdbc:mysql://localhost:3306/test_bd>" \
      --username=<username> \
      --password=<password> \
      --diffTypes="tables, views, columns, indexes, foreign keys, primary keys, unique constraints, data" 
      generateChangeLog
```

When we have a generated file `changelogs.xml`, we can restore our database structure from that file.
The command bellow creates the tables, indexes etc. in postgresql.

```shell
sh liquibase.sh \
      --driver=org.postgresql.Driver \
      --classpath=<path to jdbc driver>\
      --changeLogFile=<path to some directory>/changelogs.xml \
      --url="jdbc:postgresql://localhost:5432/test_bd" \
      --username=<username> \
      --password=<password> \
      update
```

You should keep in your mind that new database sequences does not keep previous values, and we have to set them manually.

```sql
CREATE OR REPLACE FUNCTION setval_schema(schema_name name, raise_notice boolean = false)
    RETURNS VOID AS
-- Sets all the sequences in the schema "schema_name" to the max(id) of every table
$BODY$
 
DECLARE
    row_data RECORD;
    sql_code TEXT;
 
BEGIN
    IF ((SELECT COUNT(*) FROM pg_namespace WHERE nspname = schema_name) = 0) THEN
        RAISE EXCEPTION 'The schema "%" does not exist', schema_name;
    END IF;
 
    FOR sql_code IN
        SELECT 'SELECT SETVAL(' ||quote_literal(N.nspname || '.' || S.relname)|| ', MAX(' ||quote_ident(C.attname)|| ') ) FROM ' || quote_ident(N.nspname) || '.' || quote_ident(T.relname)|| ';' AS sql_code
            FROM pg_class AS S
            INNER JOIN pg_depend AS D ON S.oid = D.objid
            INNER JOIN pg_class AS T ON D.refobjid = T.oid
            INNER JOIN pg_attribute AS C ON D.refobjid = C.attrelid AND D.refobjsubid = C.attnum
            INNER JOIN pg_namespace N ON N.oid = S.relnamespace
            WHERE S.relkind = 'S' AND N.nspname = schema_name
            ORDER BY S.relname
    LOOP
        IF (raise_notice) THEN
            RAISE NOTICE 'sql_code: %', sql_code;
        END IF;
        EXECUTE sql_code;
    END LOOP;
END;
$BODY$
LANGUAGE 'plpgsql' VOLATILE;

SELECT setval_schema('public');
```