结果集映射
==============================

将查询结果映射到对象的过程称为结果集映射。结果集映射的目的是将查询结果映射到对象，以便于后续的处理。

我们先定义好需要查询的mapper和实体：

.. code-block:: xml

    <mapper namespace="main">
        <select id="SelectUser">
            select id, name, age from user
        </select>
    </mapper>

.. code-block:: go

    type User struct {
        Id   int64
        Name string
        Age  int
    }

juice框架提供了三种结果集映射的方式：

sql.Rows
----------------

``sql.Rows`` 是database/sql包中的一个结构体，它可以被定义为下面的 ``Rows`` 接口：

.. code-block:: go

    type Rows interface {
        Columns() ([]string, error)
        Close() error
        Next() bool
        Scan(dest ...interface{}) error
    }


如果你熟悉database/sql包，那么你应该知道，``sql.Rows`` 是一个迭代器，它的 ``Next`` 方法用于遍历查询结果，``Scan`` 方法用于将查询结果映射到对象。

我们使用 ``engine`` 来查询

.. code-block:: go

    rows, err := engine.Object("main.SelectUser").Query(nil)
    if err != nil {
        panic(err)
    }
    defer rows.Close()

    for rows.Next() {
        var user User
        err := rows.Scan(&user.Id, &user.Name, &user.Age)
        if err != nil {
            panic(err)
        }
        fmt.Println(user)
    }

    err = rows.Err()
    if err != nil {
        panic(err)
    }

Object
""""""

``Object`` 用来指定我们要执行的mapper, 它接受一个interface{}类型的参数, 它可以是以下几种类型

* ``StatementIDGetter`` 类型，它有一个 ``StatementID`` 方法，返回一个字符串，该字符串就是对应的id

  .. code-block:: go

      // StatementIDGetter is an interface for getting statement id.
      type StatementIDGetter interface {
          // StatementID returns a statement id.
          StatementID() string
      }


* ``string`` 类型，表示mapper的id

  .. code-block:: go

     engine.Object("main.SelectUser")

* 函数类型, juice 内部会去获取这个函数在代码里面的位置作为对应的id，例如在传入的是 ``main`` 包下的 ``SelectUser`` 函数，那么id就是 ``main.SelectUser``

  如果这个函数是某个自定义类型的方法，那么id就是这个自定义类型的包名.类型名.方法名（注意区分interface和struct）


.. attention::
    这里介绍的 ``Object`` 是 ``engine`` 的 ``Object`` ，下面几种方式的 ``Object`` 的作用其实是一样的，就不一一介绍了。

Executor
""""""

调用完 ``Object`` 方法后，它会返回一个 ``Executor`` 对象。``Executor`` 的定义如下：

.. code-block:: go

    // Executor is an executor of SQL.
    type Executor interface {
        Query(param interface{}) (*sql.Rows, error)
        QueryContext(ctx context.Context, param interface{}) (*sql.Rows, error)
        Exec(param interface{}) (sql.Result, error)
        ExecContext(ctx context.Context, param interface{}) (sql.Result, error)
        Statement() *Statement
    }

* ``Query`` : 接受一个参数，执行查询操作，返回 ``sql.Rows`` 对象和 ``error``

* ``QueryContext`` : 接受一个 ``context.Context`` 和一个参数，执行查询操作，返回 ``sql.Rows`` 对象和 ``error``

* ``Exec`` : 接受一个参数，执行非查询操作，返回 ``sql.Result`` 对象和 ``error``

* ``ExecContext`` : 接受一个 ``context.Context`` 和一个参数，执行非查询操作，返回 ``sql.Result`` 对象和 ``error``

* ``Statement`` : 返回当前的statement对象

因为我们这里是查询操作，所以我们使用 ``Query`` 方法，并且我们的sql语句没有参数，所以我们传入 ``nil``

得到 ``sql.Rows`` 后，我们可以使用 ``sql.Rows`` 的方法来遍历查询结果，最后关闭 ``sql.Rows``。

这种方式跟database/sql包的使用方式是一样的，所以如果你熟悉database/sql包，那么你应该很容易上手。

BinderManager
---------------

BinderManager是一个接口类型，它的定义如下

.. code-block:: go

    type BinderManager interface {
        Object(v any) BinderExecutor
    }

它只有一个 ``Object`` 方法，它接受一个参数，返回一个 ``BinderExecutor`` 对象。

其中 ``Object`` 方法的作用跟上面的是一样的，用来指定查询的mapper，这里就不再介绍了。

BinderExecutor
""""""""""""""

``BinderExecutor`` 是一个接口类型，它的定义如下

BinderExecutor是一个接口类型

.. code-block:: go

    // BinderExecutor is a binder executor.
    // It is used to bind the result to the given value.
    type BinderExecutor interface {
        Query(param any) (Binder, error)
        QueryContext(ctx context.Context, param any) (Binder, error)
        Exec(param any) (sql.Result, error)
        ExecContext(ctx context.Context, param any) (sql.Result, error)
    }

BinderExecutor 跟上面的Executor的区别在于，它的 ``Query`` 方法返回的是一个 ``Binder`` 对象，而不是 ``sql.Rows`` 对象。

Query接受的参数依然是我们需要传递给mapper的参数，如果没有参数，那么传入 ``nil`` 即可。

Binder
""""""

``Binder`` 是一个接口类型，它的定义如下

Binder是一个接口类型

.. code-block:: go

    // Binder bind sql.Rows to dest
    type Binder interface {
        // Scan sql.Rows to dest
        // dest can be a pointer to a struct, a pointer to a slice of struct, or a pointer to a slice of any type.
        Scan(v any) error
    }

``Scan`` 方法接受一个参数，这个参数可以是一个指向结构体的指针，也可以是一个指向结构体切片的指针，也可以是一个指向任意类型切片的指针。

具体用法可以参考下面的例子:

.. code-block:: go

    binder, err := juice.NewBinderManager(engine).Object("main.SelectUser").Query(nil)
    if err != nil {
        panic(err)
    }
    var users []User
    if err = binder.Scan(&users); err != nil {
        panic(err)
    }
    fmt.Println(users)

因为我们查询集是一个list，所以我们传入一个指向User切片的指针，然后调用 ``Scan`` 方法，将查询结果绑定到切片中。

但是我们一运行，发现我们的users切片有元素，但是结构体全部都是零值结构体，这是为什么呢？

这是因为我们的User结构体中的字段名和数据库中的字段名不一致，所以在结构体字段上指定数据库的字段名即可。

我们修改一下User结构体

.. code-block:: go

    type User struct {
        Id   int64  `column:"id"`
        Name string `column:"name"`
        Age  int    `column:"age"`
    }

我们使用 ``column`` 标签指定了数据库中的字段名，然后再次运行，发现我们的users切片中的元素已经有值了。


GenericManager
---------------

GenericManager是一个接口类型，它的定义如下

.. code-block:: go

    type GenericManager[T any] interface {
        Object(v any) GenericExecutor[T]
    }

它只有一个 ``Object`` 方法，它接受一个参数，返回一个 ``GenericExecutor`` 对象。

其中 ``Object`` 方法的作用跟上面的是一样的，用来指定查询的mapper，这里就不再介绍了。

注意的是，这里的 ``GenericManager`` 需要接受一个泛型参数，这个参数用来指定 ``GenericExecutor`` 的返回值类型，也就是我们的查询结果类型。

GenericExecutor
"""""""""""""""

``GenericExecutor`` 是一个接口类型，它的定义如下

.. code-block:: go

    // GenericExecutor is a generic executor.
    type GenericExecutor[result any] interface {
        Query(param any) (result, error)
        QueryContext(ctx context.Context, param any) (result, error)
        Exec(param any) (sql.Result, error)
        ExecContext(ctx context.Context, param any) (sql.Result, error)
    }


它的 ``Query`` 方法返回我们指定的查询结果类型和一个 ``error`` ，而不是 ``Binder`` 对象。

使用示例

.. code-block:: go

    users, err := juice.NewGenericManager[[]User](engine).Object("main.SelectUser").Query(nil)
    if err != nil {
        panic(err)
    }
    fmt.Println(users)

这里我们使用 ``NewGenericManager`` 方法创建了一个 ``GenericManager`` 对象，因为我们的查询结果是一个list，所以我们指定这个对象的泛型参数是 ``[]User`` ，也就是说它的返回值类型是 ``[]User`` ，然后我们调用 ``Object`` 方法指定查询的mapper，然后调用 ``Query`` 方法执行查询，最后返回一个 ``[]User`` 类型的结果。

同样的，我们需要在User结构体上指定数据库字段名。


ResultMap
---------
