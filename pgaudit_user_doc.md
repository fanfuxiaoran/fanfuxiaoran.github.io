# 初识 pgaudit

## pgaudit 官方介绍
The PostgreSQL Audit Extension (pgAudit) provides detailed session and/or object audit logging via the standard PostgreSQL logging facility.

The goal of the pgAudit is to provide PostgreSQL users with capability to produce audit logs often required to comply with government, financial, or ISO certifications.

An audit is an official inspection of an individual's or organization's accounts, typically by an independent body. The information gathered by pgAudit is properly called an audit trail or audit log. The term audit log is used in this documentation.

## pgaudit 日志的格式

- AUDIT_TYPE - SESSION or OBJECT 
- STATEMENT_ID - ID of the statement in the current backend 
- SUBSTATEMENT_ID - ID of the substatement in the current backend 
- CLASS - Class of statement being logged (e.g. ROLE, READ, WRITE/DDL/FUNCTION/MISC/MISC_SET) 
- COMMAND - e.g. SELECT, CREATE ROLE, UPDATE 
- OBJECT_TYPE - When available, type of object acted on (e.g. TABLE, VIEW) 
- OBJECT_NAME - When available, fully-qualified table of object 
- STATEMENT - The statement being logged 
- PARAMETER - If parameter logging is requested, they will follow the statement 

## Examples:
```
WARNING:  AUDIT: SESSION,1,1,READ,SELECT,,,"SELECT * FROM test3, test2;",<not logged 

NOTICE:  AUDIT: SESSION,13,1,ROLE,GRANT,TABLE,,"GRANT SELECT ON vw_test3 TO auditor;",<not logged> 

NOTICE:  AUDIT: SESSION,4,1,DDL,CREATE TABLE,TABLE,public.tmp,"CREATE TABLE tmp (id int, data text);",<not logged> 

NOTICE:  AUDIT: SESSION,83,1,MISC,SET,,,SET search_path TO public;,DEALLOCATE pgclassstmt; 

NOTICE:  AUDIT: SESSION,13,1,MISC,DEALLOCATE,,,DEALLOCATE pgclassstmt;,<none> 
```

## pgaudit 生成日志的时机(AUDIT TYPE) 
### Session log
可以在一个session中设置pgaudit.log 为下面的多种选择的组合：

LOG_DDL/ LOG_FUNCTION/ LOG_MISC/ LOG_READ/ LOG_ROLE/ LOG_WRITE/ LOG_MISC_SET

还有另外两个选项LOG_NONE和LOG_ALL

如果设置成LOG_NONE则该session不会生成任何pgaudit相关日志，设置成LOG_ALL则会针对上述类型的操作都会生成日志,
默认值是LOG_NONE。

对于session的log，还有两个GUC:pgaudit.log_relation 和pgaudit.log_catalog. 

如果pgaudit.log_relation 设置为true，则会对每一条log涉及到的每个relation单独生成一条log

如果pgaudit.log_catalog设置为false, 则对于pg_catalog的表不生成日志

### Examples:
```
SELECT 1,   substring('Thomas' from 2 for 3);

NOTICE:  AUDIT: SESSION,28,1,READ,SELECT,,,"SELECT 1,substring('Thomas' from 2 for 3);",<none>

SELECT * FROM test3, test2; (log_relation 是false)

WARNING:  AUDIT: SESSION,1,1,READ,SELECT,,,"SELECT *  FROM test3, test2;",<not logged>
```

### Object log (role based log)

对应的GUC是pgaudit.role:

  Specifies the master role to use for object audit logging.  Multiple audit roles can be defined by granting them to the master role. This allows multiple groups to be in charge of different aspects of audit logging

如果想log某个relation的读写，可以把读写权限赋予pgaudit.role，也可以只把relation中的某些列的读写权限赋予pgaudit.role, 这样在对该relaiton进行读写的时候，pgaudit会检查pgaudit.role的权限，对于有权限的操作产生日志。

If a privilege has been granted on an entire table, it will show up in this view as a grant for each column, but only for the privilege types where column granularity is possible: SELECT, INSERT, UPDATE, REFERENCES.
https://www.postgresql.org/docs/current/infoschema-column-privileges.html

session log 和 object log 可以混合使用

## pgaudit log哪些东西
- 表数据的读写 
- ddl 操作
- log_role 权限变更的操作
- 其它：function、extension 、misc(set等)

## Examples:

```
CREATE TABLE tmp2 AS (SELECT * FROM tmp);

NOTICE:  Table doesn't have 'DISTRIBUTED BY' clause. Creating a NULL policy entry.

NOTICE:  AUDIT: SESSION,6,1,READ,SELECT,TABLE,public.tmp,CREATE TABLE tmp2 AS (SELECT * FROM tmp);,<not logged>

NOTICE:  AUDIT: SESSION,6,1,WRITE,INSERT,TABLE,public.tmp2,CREATE TABLE tmp2 AS (SELECT * FROM tmp);,<not logged>

NOTICE:  AUDIT: SESSION,6,2,DDL,CREATE TABLE AS,TABLE,public.tmp2,CREATE TABLE tmp2 AS (SELECT * FROM tmp);,<not logged>
```





