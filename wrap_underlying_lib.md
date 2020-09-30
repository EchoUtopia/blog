## 食用库之前先为它裹上糖衣

以标准库中的数据库客户端为例: `*sql.DB`

我们可以自己封装一层：

```go
type DB struct {
    *sql.DB
    your fields
    ...
}
```

这样，我们就可以为DB增加一些按分层设计应该由DB来提供的方法，来扩展这些我们不能控制的底层库的功能。

我们以sqlx这个库为例, sqlx.DB结构体为：

```go
type DB struct {
	*sql.DB
	driverName string
	unsafe     bool
	Mapper     *reflectx.Mapper
}
```

sqlx提供了标准库db以外很多很强大很好用的功能，例如：
- 可以帮我们隐藏一些不同数据库的差异，让我们一套代码可以兼容多个数据库。
- sqlx提供了很方便的查询的方法。

当我们使用sql.DB查询一个数据列表时，我们需要写很啰嗦的代码

```go
query := `****`
result := []type{}
rows, err := db.Query(query, args...)
handleErr(err)
defer rows.Close()
for rows.Next(){
    item := new(***)
    if err := rows.Scan(&item.A, &item.B)；err != nil {
        handleErr(err)
    }
    result = append(result, item)
}
```

如果我们使用sqlx.DB,就可以很简洁的代码把数据查出来：

```go
query := `****`
result := []type{}
if err := db.Select(&result, query, args...);err != nil {handleErr(err)}

```
在我的项目中，我将db常用的查询方法拆分出了interface，例如：



```go

type DBer interface {
	Exec(string, ...interface{}) (sql.Result, error)
	Query(string, ...interface{}) (*sql.Rows, error)
	QueryRow(string, ...interface{}) *sql.Row
}

type SQLXDBer interface {
	DBer
	Rebind(query string) string
	Select(dest interface{}, query string, args ...interface{}) error
	Get(dest interface{}, query string, args ...interface{}) error
}

type SQLXTxer interface {
	SQLXDB
	Commit() error
	Rollback() error
	RegisterAfterCommitHooks(fns ...func() error)
}
```


由sql.DB拆分出来的查询interface为DBer，

由sqlx.DB拆分出来的查询interface为SQLXDBer

当我们使用数据库操作时，我们可以自己封装interface，这样，只要满足接口操作我们就可以实现数据库操作，不管底层是不是标准库实现。


下面我举两个我在实际项目中对数据库扩展的功能：

#### 自动翻译postgres的sql语句到oracle的语句

我们很多项目数据库使用的postgres，也没有用orm，突然有一天新客户要求项目需要支持oracle，但是呢，将所有sql翻译一遍成本太高，

我便写了个基础库，将postgres的sql语句翻译成oracle的语句。

为了让上层无感知实现自动翻译，我定义了一下结构体：
```go
type Translator struct {
	Driver           string
	DB               SQLXDBer
	logger           XOLogger
	afterCommitHooks []func() error
}

func (t *Translator) Exec(query string, args ...interface{}) (sql.Result, error) {
	var err error
	if t.Driver != DriverPg {
		query, args, err = t.translate(query, args...)
		if err != nil {
			return nil, err
		}
	}
	return t.DB.Exec(query, args...)
}
。。。
```
Translator既实现了SQLXDB， 又实现了SQLXTxer，查询数据时，最终会执行Translator的方法，

我会判断driver是不是postgres，如果是oracle，就翻译sql。


#### 为数据库操作增加钩子函数

这里说两句题外话：

在我设计业务逻辑时，我会对各种逻辑进行分层，model在最底层。
只不过我的逻辑也会分层，底层是一堆比较简单的逻辑，它们负责的逻辑单元比较小，一旦确定了基本上不会怎么变，是顶层逻辑的基石。

然后上层做的就是对这些底层逻辑单元进行组合。

当业务流程比较繁杂的，我会视情况多分几层。

最终就像一棵树一样，对用户暴漏的一个接口就像树的根节点，然后一直往下延伸形成一棵树。

底层的一个操作数据库的函数，它接受SQLXDBer，它既可以是在事务中执行，也可以单独执行，rollback由顶层的业务流程来操作，底层就不管事务这一概念了，专注于他自己的逻辑就好了。

如果只能在事务中执行，那参数类型可以指定为SQLXTxer。



SQLXTxer实现了RegisterAfterCommitHooks，给afterCommit事件添加了钩子函数，主要为了实现以下功能：

在业务逻辑中，有些对象有缓存，当更改关于这些对象的数据时，需要删除对象的缓存（或者更新）

这些更改可能在事务中，那就需要在事务提交以后再删除缓存，否则，在删除缓存后事务提交前，其他事务可能会读到老数据并更新缓存，导致缓存里是老数据，这在很多场景是不可接受的。

在业务中，事务提交在顶层逻辑流程中，如果需要顶层来删缓存，那么会导致底层的逻辑被扩散到顶层，从代码架构来说不可接受。

所以，在底层操作数据时，注册commit后的钩子函数，来删除缓存，这样当顶层提交事务时，自动就会删缓存，这个过程对顶层不可见，代码结构就比较清晰明了了。