事务
========

我们在操作数据库的时候，经常会遇到事务的问题，比如说，我们在操作数据库的时候，需要对多个表进行操作，这时候，我们就需要使用事务来保证数据的一致性。

那么在 ``juice`` 中，我们是如何使用事务的呢？

首先我们回顾一下我们之前的用法:

.. code-block:: go

	cfg, err := juice.NewXMLConfiguration("config.xml")

	if err != nil {
		panic(err)
	}

	engine, err := juice.Default(cfg)

	if err != nil {
		panic(err)
	}

	engine.Object("your object").Query(nil)

	juice.NewGenericManager[User](engine).Object("your object").QueryContext(context.TODO(), nil)


在上面的代码中，我们可以看到，我们都是通过 ``engine`` 来进行操作的，``engine`` 会从数据库的连接池中获取一个连接，然后进行操作，操作完毕后，会将连接放回连接池中。

那么，如果我们需要使用事务，我们该如何操作呢？

Manager
-------

通过查看 ``NewGenericManager`` 的源码，我们可以看到，它接受的参数都是一个名为 ``Manager`` 的接口。

Manager 接口的定义如下:

.. code-block:: go

	// Manager is an interface for managing database operations.
	type Manager interface {
		Object(v any) Executor
	}

我们上面使用的 ``engine`` 就是一个实现了 ``Manager`` 接口的结构体。

开启事务
--------

我们可以通过 ``engine.Tx()`` 来开启一个事务，它会返回一个 ``Manager`` 接口的实现，我们可以通过这个实现来进行事务的操作。

注意，当调用 ``engine.Tx()`` 事务就已经开启了。

我们可以通过 ``engine.Tx()`` 返回的事务对象来进行事务的操作。

如上面的示例我们可以这样写:

.. code-block:: go

	cfg, err := juice.NewXMLConfiguration("config.xml")

	if err != nil {
		panic(err)
	}

	engine, err := juice.Default(cfg)

	if err != nil {
		panic(err)
	}

	tx := engine.Tx()

	defer tx.Rollback()

	{
		tx.Object("your object").QueryContext(context.TODO(), nil)
	}

	{
		juice.NewGenericManager[User](tx).Object("your object").QueryContext(context.TODO(), nil)
	}

	tx.Commit()

在上面的示例中，我们可以看到，我们将 ``Manager`` 由 ``engine`` 改成 ``tx`` 之后，上面的操作都会在一个事务中进行了，我们就通过 ``tx`` 来进行事务的操作，当我们需要提交事务的时候，我们调用 ``tx.Commit()`` 来提交事务，如果我们需要回滚事务，我们调用 ``tx.Rollback()`` 来回滚事务。

注意，当我们调用 ``tx.Commit()`` 或者 ``tx.Rollback()`` 之后，事务就已经结束了，我们不能再对事务进行操作了。

如果你需要使用嵌套事务，多次调用 ``engine.Tx()`` 来开启事务，但是需要注意的是，你需要在每次调用 ``engine.Tx()`` 之后，都要调用 ``tx.Rollback()`` 或者 ``tx.Commit()`` 来终止事务，否则你的事务会一直处于开启状态。


隔离级别
----------

在go官方的database/sql包里面提供了对事务隔离级别的控制

.. code-block:: go

	// IsolationLevel is the transaction isolation level used in TxOptions.
	type IsolationLevel int

	// Various isolation levels that drivers may support in BeginTx.
	// If a driver does not support a given isolation level an error may be returned.
	//
	// See https://en.wikipedia.org/wiki/Isolation_(database_systems)#Isolation_levels.
	const (
		LevelDefault IsolationLevel = iota
		LevelReadUncommitted
		LevelReadCommitted
		LevelWriteCommitted
		LevelRepeatableRead
		LevelSnapshot
		LevelSerializable
		LevelLinearizable
	)

	// TxOptions holds the transaction options to be used in DB.BeginTx.
	type TxOptions struct {
		// Isolation is the transaction isolation level.
		// If zero, the driver or database's default level is used.
		Isolation IsolationLevel
		ReadOnly  bool
	}

	func (db *DB) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error) 


在juice中也提供了这样的功能

.. code-block:: go

	func (e *Engine) ContextTx(ctx context.Context, opt *sql.TxOptions) TxManager