# 浅谈postgres object_access_hook

object_access_hook是让人既爱又恨，它很万能、灵活，但是用起来很麻烦，容易踩坑。
由于某些原因，最近在使用object_access_hook遇到一些问题，记录一下。

## 初识 object_access_hook

Object access hooks are intended to be called just before or just after performing certain actions on a SQL object.  This is intended as infrastructure for security or logging pluggins.

简单理解就是会在更改SQL object之前或者之后，会调用object_access_hook 函数, 根据调用时间、操作、object类型分为几类：

OAT_POST_CREATE、OAT_DROP、OAT_POST_ALTER、 OAT_NAMESPACE_SEARCH、OAT_FUNCTION_EXECUTE、OAT_TRUNCATE

### 什么是object?

数据库的object包含table、view、function、index、 procedures, and operators、Data types and domains、Triggers and rewrite rules.
上述是一个数据库内部的object，数据库的object还包含全局的role、tablespace、database这些对象。

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
可以InvokeObjectxxxHook嵌套在pg很多地方。
## 使用object_access_hook一些问题

### 翻译Object

通过getObjectIdentity 可以获取object的identity(string)也就是名字，例如表的名字 “foo”

通过getObjectTypeDescription可以获取object的type(string),就是object类型的名字，例如table，view等

上述两个函数都调用了getObjectClass,  该函数根据 object->classId 返回一个ObjectClass （enum）.

关于objectSubId, 可以是一个表中某一列的位置，比如 table test(a int, b int), 如果objectSubId是1， 则表示列“a”。 目前只看到objectSubId用来表示某列。

这里有一个坑需要注意：当 ObjectType 是” OCLASS_DEFAULT”时候，object->classId 是“AttrDefaultRelationId”， 但是object->objectId不是tuple OID, 是relation OID.
### 可见性问题

可见性问题是使用object_access_hook最大的阻碍。
InvokeObjectPostCreateHook在object 创建之后, CommandCounterIncrement()之前执行
InvokeObjectDropHook 在 object drop 之后，CommandCounterIncrement()之前执行
InvokeObjectPostAlterHook 在object更改之后, CommandCounterIncrement()之前执行

其余的同样都是在CommandCounterIncrement执行之前调用的object_access_hook函数，因此如果在object_access_hook使用snapshot_mvcc是查询不到对应的object的信息的。因此需要使用 SnapshotSelf。

使用SnapshotSelf会引出另外的一个问题。“getObjectIdentity” 获取object identity 和“getObjectTypeDescription”  获取object type时候都是通过relation_open 或者　system cache 获取catalog表里的内容，但是cache里面的东西都是使用的CalogSnapshot，CalogSnapshot的类型是SNAPSHOT_MVCC。(此处不理解的话可以去了解pg的cache及snapshot模块)， 因此如果使用snapshotSelf的话，不能使用relation_open，只能自己手动设置snapshot 并调用“systable_beginscan” 去磁盘扫描表。不使用cache会有一定的性能损失，但是没办法

### Delete objects有依赖问题

上一部分讲到使用sanpshot_self解决可见性的问题，但是snapshot_self 用在"OAT_DROP" 的hook上就会出问题，因为存在循环依赖问题。
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

怎么解决该问题呢？还是通过更改snapshot，在drop命令执行之前获取一个snapshot（主要是获取commandId）,  在object_access_hook里使用它。

### 全局变量+ 递归调用的问题

使用hook的时候避免不了使用全部变量，并涉及到memory context，因为object_access_hook的参数是固定的，没办法再传递其他变量。

使用全局变量的时候需要注意，pg执行sql的时候会递归执行，因此一个object_access_hook会被多次调用(参考ProcessUtilityContext)，需要注意全局变量的修改及memory context问题。



oooo
