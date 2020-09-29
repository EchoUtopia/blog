## 封装底层库

举例：

type XODB interface {
	Exec(string, ...interface{}) (sql.Result, error)
	Query(string, ...interface{}) (*sql.Rows, error)
	QueryRow(string, ...interface{}) *sql.Row
}

type SQLXDB interface {
	XODB
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


当我们使用数据库操作时，我们可以自己封装interface，这样，只要满足接口操作我们就可以实现数据库操作，不管底层是不是标准库实现。


这样我们可以做很多自己的操作。

比如在我的项目中，我们需要支持oracle操作，我便写了个基础库，将postgres的sql语句翻译成oracle的语句，这样，我就可以不用*sql.Db或者*sql.Tx了，我可以封装一个自己的结构体：

type Translator struct {
	Driver           string
	DB               SQLXDB
	logger           XOLogger
	committer        Committer
	afterCommitHooks []func() error
}

Translator既实现了SQLXDB， 又实现了SQLXTxer，当实现这两个的interface方法时，我会判断driver是不是postgres，如果是oracle，就翻译sql。



在我设计业务逻辑时，我会对各种逻辑进行分层，就比如mvc一样，model在最底层。
只不过我的逻辑也会封层，底层是一堆零散的逻辑，它们负责的逻辑单元比较小，一旦确定了基本上不会怎么变，是顶层逻辑的基石。

然后上层做的就是对这些底层逻辑单元进行组合。

当业务流程比较繁杂的，我会视情况多分几层。

最终就像一棵树一样，对用户暴漏的一个接口就像树的根节点，然后一直往下延伸形成一棵树。

比如底层的一个操作数据库的函数，它接受SQLXDB，他既可以时在事务中执行，也可以单独执行，rollback由顶层的业务流程来操作，底层就不管事务这一概念了，如果只能在事务中执行，那参数类型就是SQLXTxer了。

另外，可以看到SQLXTxer实现了RegisterAfterCommitHooks，这里是为了更好的实现自己的逻辑。

在业务逻辑中，有些对象有缓存，当更改关于这些对象的数据时，缓存也需要对应更改。

这些更改可能在事务中，那就需要在事务提交以后再删除缓存，否则，如果事务还没提交就删除缓存，在删除缓存后事务提交前，其他事务可能会读到老数据并更新缓存，导致缓存里是老数据，这在很多场景是不可接受的。在业务中，事务提交在顶层逻辑流程中，如果需要顶层来删缓存，那么会导致底层的逻辑被扩散到顶层，从代码架构来说不可接受。

所以，在底层操作数据时，注册commit后的钩子函数，来删除缓存，这样当顶层提交事务时，自动就会删缓存，这个过程对顶层不可见，代码结构就比较优良了。