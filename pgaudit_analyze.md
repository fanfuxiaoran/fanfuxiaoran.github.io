# pgaudit 源代码重点解析

(参考 branch REL_12_STABLE)

pgaudit 的介绍和使用参 [Postgres pgaudit 介绍](pgaudit_user_doc.md)

pgaudit 就是要在 DML 过程中收集数据对象的读写信息，在DDL执行过程中收集数据对象的更改信息。

DML
```
INSERT INTO test3 (id, name)
                  VALUES (1, 'a');
NOTICE:  AUDIT: SESSION,16,1,WRITE,INSERT,TABLE,public.test3,"INSERT INTO test3 (id, name)
                  VALUES (1, 'a');",<none>,1
```

DDL
```
CREATE TABLE public.test
(
        id INT
);
NOTICE:  AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.test,"CREATE TABLE public.test
(
        id INT
);",<not logged>
```

Postgres 本身没有提供该功能，但是提供了一些基础的 hook 及 DDL event collect 机制，可以利用它们实现该功能。

## DDL event collect 收集 DDL 过程中的 objects 信息

Postgres 中实现该 feature 的第一个 commit 是: 488c580aef4e05f39be5daaab6464da5b22a494


```
Allow on-the-fly capture of DDL event details

This feature lets user code inspect and take action on DDL events.
Whenever a ddl_command_end event trigger is installed, DDL actions
executed are saved to a list which can be inspected during execution of
a function attached to ddl_command_end.
```
它可以捕捉一条 query 执行过程中所有 DDL events，用户可以通过实现一个 EventTrigger 来获取它们。该Trigger会在 ddl_command_end 的时候执行，返回给用户所有的 DDL events。pgåudit 中 EventTrigger 实现如下：

```
CREATE FUNCTION pgaudit_ddl_command_end()
        RETURNS event_trigger
        SECURITY DEFINER
        SET search_path = 'pg_catalog, pg_temp'
        LANGUAGE C
        AS 'MODULE_PATHNAME', 'pgaudit_ddl_command_end';

CREATE EVENT TRIGGER pgaudit_ddl_command_end
        ON ddl_command_end
        EXECUTE PROCEDURE pgaudit_ddl_command_end();
```

其实现机制:

- 初始化一些状态及全区变量
```
EventTriggerDDLCommandStart(parsetree);
```

- 开始收集ddl event
```
address = DefineView((ViewStmt *) parsetree, queryString, pstmt->stmt_location, pstmt->stmt_len);
EventTriggerCollectSimpleCommand(address, secondaryObject, parsetree);
```
先DefineView, 再调用 EventTriggerCollectSimpleCommand 收集 object 信息。

其中的细节不展开来讲了：比如在一个command的执行过程中涉及到多个 object，为保证 object 收集到的先后顺序是正确的，EventTriggerCollectSimpleCommand 会穿插到很多地方执行。及拿到objects后对object的翻译等细节。

## 生成DML过程中objects 读写信息

通过hook` ExecutorCheckPerms_hook` 实现。

Postgres中在对object进行读写的时候都会调用该hook. 该hook的调用是在function "ExecCheckPermissions" 里面，来看一下
"ExecCheckPermissions" 调用

```
src/backend/utils/adt/ri_triggers.c
src/backend/executor/execMain.c
src/backend/commands/copy.c
```
## 记录 object 对应的上下文

由前面两个步骤可以拿到 object的类型、名称，除此之外还需要记录一些其它的 context 信息，例如对应的 sql statement, command 的类型等

```
CREATE TABLE tmp2 AS (SELECT * FROM tmp);
NOTICE:  AUDIT: SESSION,5,1,READ,SELECT,,,CREATE TABLE tmp2 AS (SELECT * FROM tmp);,<not logged>
NOTICE:  AUDIT: SESSION,5,2,DDL,CREATE TABLE AS,TABLE,public.tmp2,CREATE TABLE tmp2 AS (SELECT * FROM tmp);,<not logged>
```
其中第一条作所属于的 sql 是 "CREATE TABLE tmp2 AS (SELECT * FROM tmp)", command 是 select,  class 是 read

可以看出，一条 sql 语句不止涉及到一个 object 的变动。

### pgaudit 是怎么记录 object 对应的 context 信息的呢？

在 pg 中, sql 语句执行有两个入口，ddl 的入口是 "ProcessUtility",  dml 的入口是 "ExecutorStart"， 这两个入口都有对应的 hook， 分别是 "ProcessUtility_hook" 和 "ExecutorStart_hook"。 可以在这两个入口处记录对应的 sql 语句及其它的 context 信息即可。

还有另外的一个问题，就是用户输入的一条 sql 语句，可能在 pg server 端拆成几个 command 来执行，command 之间大多是嵌套执行的。

CREATE TABLE AS

还是上面的 "CREATE TABLE tmp2 AS (SELECT * FROM tmp);" 这条sql的执行路径

1.	是 dml ， 进入ProcessUtility-> ProcessUtilitySlow
2.	调用 ExecCreateTableAs，并解析 IntoClause 生成 DestReceiver 描述。 检查对应的 query(select * from tmp) ，解析生成 plan。
3.	执行第二部的生成的 plan ,调用 ExecutorStart，调用 ExecutorStart_hook
4.	调用 intorel_startup，创建目的表 tmp2.


因为不同的 command 嵌套执行，因此 pgaudit 使用栈来记录 context 信息。在ProcessUtility  和 ExecutorStart 这两个入口处分别push一个 记录 context 信息的 AuditItem。

ProcessUtility 执行过程中，所涉及的 objects 会收集到一个链表中，在 ddl_command_end 的时候通过 trigger 获取并一起打印

ExecutorStart  执行过程中，是见到一个 object 便直接打印

由此可见，虽然不同的 command 的对应的 object 变化是相互交叉在一起的，但是由于 ProcessUtility 先收集再打印，因此不会引起混乱。

AuditItem 什么时候出栈呢？

通过 MemoryContextCallBack 机制实现。每个query 执行的时候都有对应 Memory context, 把 AuditItem 的memory context挂到 query 执行的 memory context 下，当 query 执行完成时，会 reset 或者 delete memory context, 这时候调用 call back 函数出栈








