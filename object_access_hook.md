# 浅谈postgres object_access_hook

object_access_hook 是让人既爱又恨，它很万能、灵活，但是用起来很麻烦，容易踩坑。
由于某些原因，最近在使用 object_access_hook 遇到一些问题，记录一下。

## 初识 object_access_hook

Object access hooks are intended to be called just before or just after performing certain actions on a SQL object.  This is intended as infrastructure for security or logging pluggins.

简单理解就是会在更改 object 之前或者之后，调用 object_access_hook 函数, 根据调用时间、操作、object 类型分为几类：

OAT_POST_CREATE、OAT_DROP、OAT_POST_ALTER、 OAT_NAMESPACE_SEARCH、OAT_FUNCTION_EXECUTE、OAT_TRUNCATE

### 什么是object?

数据库的 object 包含 table、view、function、index、 procedures, and operators、Data types and domains、Triggers and rewrite rules.
上述是一个数据库内部的 object，数据库的 object 还包含全局的 role、tablespace、database 这些对象。

Object的内部表示
```
/*
 * An ObjectAddress represents a database object of any type.
 */
typedef struct ObjectAddress
{
        Oid                     classId;                /* Class Id from pg_class */
        Oid                     objectId;               /* OID of the object */
        int32           objectSubId;    /* Subitem within object (eg column), or 0 */
} ObjectAddress;
```

这里来看一下 InvokeObjectDropHook 调用的地方

```
src/backend/catalog/dependency.c
1303: deleteOneObject （删除数据库内的一个对象) -> InvokeObjectDropHookArg(object->classId, object->objectId,

src/backend/commands/subscriptioncmds.c
1530: DropSubscription -> InvokeObjectDropHook(SubscriptionRelationId, subid, 0);

src/backend/commands/tablespace.c
469: DropTableSpace -> InvokeObjectDropHook(TableSpaceRelationId, tablespaceoid, 0);

src/backend/commands/user.c
1184: DropRole -> InvokeObjectDropHook(AuthIdRelationId, roleid, 0);

src/backend/commands/dbcommands.c
1633: dropdb-> InvokeObjectDropHook(DatabaseRelationId, db_id, 0);
```
可以看到InvokeObjectxxxHook嵌套在pg很多地方。

## 使用 object_access_hook 一些问题

### 翻译 object

通过 getObjectIdentity 可以获取 object 的 identity(string) 也就是名字，例如表的名字 “foo”

通过 getObjectTypeDescription 可以获取 object 的 type(string),就是 object 类型的名字，例如 table，view等

上述两个函数都调用了getObjectClass,  该函数根据 object->classId 返回一个 ObjectClass （enum）.

关于 objectSubId, 可以是一个表中某一列的位置，比如 table test(a int, b int), 如果 objectSubId 是1，则表示列“a”。 目前只看到 objectSubId 用来表示某列。

这里有一个坑需要注意：当 ObjectType 是 "OCLASS_DEFAULT" 时候，object->classId 是“AttrDefaultRelationId”， 但是object->objectId不是tuple OID, 是relation OID.
### 可见性问题

可见性问题是使用 object_access_hook 最大的阻碍。

InvokeObjectPostCreateHook 在object 创建之后, CommandCounterIncrement()之前执行

InvokeObjectDropHook 在 object drop 之后，CommandCounterIncrement()之前执行

InvokeObjectPostAlterHook 在object更改之后, CommandCounterIncrement()之前执行

其余的同样都是在 CommandCounterIncrement 执行之前调用的 object_access_hook 函数，因此如果在 object_access_hook 使用 snapshot_mvcc 是查询不到对应的 object 的信息的。因此需要使用 SnapshotSelf。

使用 SnapshotSelf 会引出另外的一个问题。“getObjectIdentity” 获取 object identity 和 “getObjectTypeDescription”  获取 object type 时候都是通过 relation_open 或者　system cache 获取 catalog 表里的内容，但是 cache 里面的东西都是使用的 CalogSnapshot，CalogSnapshot 的类型是 SNAPSHOT_MVCC。(此处不理解的话可以去了解pg的cache及 snapshot 模块)， 因此如果使用 snapshotSelf 的话，不能使用 relation_open，只能自己手动设置snapshot 并调用 “systable_beginscan” 去磁盘扫描表。不使用 cache 会有一定的性能损失，但是没办法

### Delete objects有依赖问题

上一部分讲到使用 sanpshot_self 解决可见性的问题，但是 snapshot_self 用在 "OAT_DROP" 的hook上就会出问题，因为存在循环依赖问题。
例子：
```
    CREATE TYPE casttesttype;

    CREATE FUNCTION casttesttype_in(cstring)
       RETURNS casttesttype
       AS 'textin'
       LANGUAGE internal STRICT;
    CREATE FUNCTION casttesttype_out(casttesttype)
       RETURNS cstring
       AS 'textout'
       LANGUAGE internal STRICT;

    CREATE TYPE casttesttype (
       internallength = variable,
       input = casttesttype_in,
       output = casttesttype_out,
       alignment = int4
    );

    DROP FUNCTION casttesttype_in(cstring) cascade;
    NOTICE:  drop cascades to 2 other objects
    DETAIL:  drop cascades to type casttesttype
    drop cascades to function casttesttype_out(casttesttype)
```
原因此处不展开细讲了

怎么解决该问题呢？还是通过更改 snapshot，在 drop 命令执行之前获取一个 snapshot（主要是获取commandId）,  在 object_access_hook 里使用它。

### 全局变量+ 递归调用的问题

使用 hook 的时候避免不了使用全部变量，并涉及到 memory context，因为 object_access_hook 的参数是固定的，没办法再传递其他变量。

使用全局变量的时候需要注意，pg执行sql的时候会递归执行，因此一个 object_access_hook 会被多次调用(参考ProcessUtilityContext)，需要注意全局变量的修改及memory context问题。

