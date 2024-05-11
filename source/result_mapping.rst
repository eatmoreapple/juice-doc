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


* ``string`` 类型，表示对于 ``action`` 的完整id。

  .. code-block:: go

     engine.Object("main.SelectUser")

* 函数类型, juice 内部会去获取这个函数在代码里面的位置作为对应的id，例如在传入的是 ``main`` 包下的 ``SelectUser`` 函数，那么id就是 ``main.SelectUser``

  如果这个函数是某个自定义类型的方法，那么id就是这个自定义类型的 ``pkgpath.(interface|struct).methodname`` （包名.类型名.方法名, 注意区分 ``interface`` 和 ``struct``）


.. attention::
    这里介绍的 ``Object`` 是 ``engine`` 的 ``Object`` ，下面几种方式的 ``Object`` 的作用其实是一样的，就不一一介绍了。

Executor
""""""

调用完 ``Object`` 方法后，它会返回一个 ``Executor`` 对象。``Executor`` 的定义如下：

.. code-block:: go

    // Executor is an executor of SQL.
    type Executor interface {
        QueryContext(ctx context.Context, param interface{}) (*sql.Rows, error)
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

泛型结果集映射
---------------

GenericManager是一个接口类型，它的定义如下

.. code-block:: go

    type GenericManager[T any] interface {
        Object(v any) GenericExecutor[T]
    }

它只有一个 ``Object`` 方法，它接受一个参数，返回一个 ``GenericExecutor`` 对象。

其中 ``Object`` 方法的作用跟上面的是一样的，用来指定查询的mapper，这里就不再介绍了。

这里的 ``GenericManager`` 需要接受一个泛型参数，这个参数用来指定 ``GenericExecutor`` 的返回值类型，也就是我们的查询结果类型。

GenericExecutor
"""""""""""""""

``GenericExecutor`` 是一个接口类型，它的定义如下

.. code-block:: go

    // GenericExecutor is a generic executor.
    type GenericExecutor[result any] interface {
        QueryContext(ctx context.Context, param any) (result, error)
        ExecContext(ctx context.Context, param any) (sql.Result, error)
    }


它的 ``Query`` 方法返回我们指定的查询结果类型和一个 ``error``。

需要注意，这里会直接将查询的结构直接映射到指定的结果集的泛型参数上面。

结果集的映射可以分为以下几种情况：

1、单个字段，单条结果。

.. code-block:: go

    // select count(*) from table

    result, err := juice.NewGenericManager[int](engine).Object("your object id").Query(nil)

    // result => int



    // select count(*) from table

    result, err := juice.NewGenericManager[[]int](engine).Object("your object id").Query(nil)

    // result => []int 有且只有一个元素



    // select c_time from table limit 1

    result, err := juice.NewGenericManager[time.Time](engine).Object("your object id").Query(nil)

    // result => time.Time

2、多个字段，单条结果。

.. code-block:: go

    type User struct {
        ID   int64  `column:"id"`
        Name string `column:"name"`
    }

    // select id, name from table limit 1

    result, err := juice.NewGenericManager[User](engine).Object("your object id").Query(nil)

    // result => User



    // select id, name from table limit 1

    result, err := juice.NewGenericManager[*User](engine).Object("your object id").Query(nil)

    // result => *User



    // select id, name from table limit 1

    result, err := juice.NewGenericManager[[]User](engine).Object("your object id").Query(nil)

    // result => []User 有且只有一个元素


    // select id, name from table limit 1

    result, err := juice.NewGenericManager[[]*User](engine).Object("your object id").Query(nil)

    // result => []*User 有且只有一个元素

*需要注意的是，当返回的字段有多条的时候，需要用结构体来接收，并且结构体字段中需要标注`column`来制定对应的字段*

*问: 可以用map来代替结构体吗？*

*答: 不可以，因为我个人不喜欢用map（开玩笑），map呈现给用户的往往是一个黑盒，开发者不知道这个map中有哪些信息，而结构体相比map而言，显的更清晰一点*

如下，当返回的字段在结构体中找不到映射怎么办？

.. code-block:: go

    type User struct {
        ID   int64  `column:"id"`
        Name string `column:"name"`
    }   

    // SELECT id, name, age from table

这种情况下，juice会将找不到字段映射的值给丢弃掉，如上面的age。

3. 单个字段，多条结果。

.. code-block:: go

    // select id from table limit 10

    result, err := juice.NewGenericManager[[]int64](engine).Object("your object id").Query(nil)

    // result => []int64

注意的是，当查询返回的行数有多条时，必须指定结果集为一个切片，否则会返回错误。

4. 多个字段，多条结果。

.. code-block:: go

    type User struct {
        ID   int64  `column:"id"`
        Name string `column:"name"`
    }   

    // select id, name from table limit 10

    result, err := juice.NewGenericManager[[]User](engine).Object("your object id").Query(nil)

    // result => []User

跟上面一样，多条查询结果需要用切片接受，多个字段需要用结构体接受。


自定义结果集映射
---------

有时候我们的映射关系比较复杂，我们需要自己去定义这种映射关系。

resultMap
``resultMap`` 是juice提供的一个自定义结果集映射规则的标签，下面我们通过几个例子来了解它的用法。

简单查询
""""""""""""""

.. code-block:: xml

    <resultMap id="User">
        <result column="id" property="ID"/>
        <result column="name" property="Name"/>
    </resultMap>

    <select id="QueryUserList" resultMap="User">
        select id, name from classes
    </select>

.. code-block:: go

    type User struct {
        ID   int64  
        Name string 
    }

    result, err := juice.NewGenericManager[[]User](engine).Object("object id").Query(nil)

当我们在 ``select`` 标签里面指定我们的resultMap的时候, juice就会采用resultMap指定的规则去进行映射。

resultMap生效的作用域在mapper标签下面。它需要一个id属性作为它的唯一身份标识符。

result标签的作用是用来指定sql字段和结构体字段的映射关系。

*   column是sql字段。
*   property是结构体映射字段名称（需要的是一个合法的可导出的字段的名称）。

一对一查询
""""""""""""""

.. code-block:: xml

    <resultMap id="User">
        <result column="id" property="ID"/>
        <result column="name" property="Name"/>
        <association property="Detail">
            <result column="id_card" property="IDCard"/>
        </association>
    </resultMap>

    <select id="QueryUser" resultMap="User">
        select a.id as id, a.name as name, b.id_card as id_card from user a join user_detail b where b.uid = a.id limit 1
    </select>

.. code-block:: go

    type UserDetail struct {
        UID    int64
        IDCard string
    }

    type User struct {
        ID         int64  
        Name       string 
        Detail     UserDetail
    }

    result, err := juice.NewGenericManager[User](manager).Object("QueryUser").Query(nil)

在resultMap中可以使用 association 来对结构体进行嵌套映射。

association需要一个property属性来制定嵌套结构体的字段名称。如上，嵌套结构体 UserDetail 在 User 中的字段名称为Detail，我们只需要在xml中将
property的属性置为Detail即可。

result标签在association中的使用方式跟在resultMap中一致。

一(多)对多查询
""""""""""""""

.. code-block:: xml

    <resultMap id="User">
        <id column="id" property="ID"/>
        <result column="name" property="Name"/>
        <collection property="Hobbies">
            <result column="hobby" property="Hobby"/>
        </collection>
    </resultMap>

    <select id="QueryUserList" resultMap="User">
        select a.id as id, a.name as name, b.hobby as hobby from user a join user_hobby b where b.uid = a.id 
    </select>

.. code-block:: go

    type Hobby struct {
        Hobby   string
    }

    type User struct {
        ID      int64
        Name    string
        Hobbies []Hobby
    }

    result, _ := juice.NewGenericManager[[]User](manager).Object("object id").Query(nil)

我们假设上面的sql的查询结果如下所示:

.. code-block:: shell

    ----------------------------
    |  id   |  name  |  hobby  |
    |---------------------------
    |   1   |  小明   |  篮球   |
    |---------------------------
    |   2   |  小李   |  唱歌   |
    |---------------------------
    |   1   |  小明   |  跳舞   |
    |---------------------------

如上所示, 我们可以很容易看出小明的hobby有两个，而小李只有一个，根据上面User结构体的定义，我们希望把第一条数据
和第三条数据组合映射到User结构体中。

如:

.. code-block:: go

    {ID: 1, Name: "小明", Hobbies: []Hobby{{"篮球"}, {"跳舞"}}}

既然需要组合，那我们得让juice知道哪些数据需要组合。

juice提供了一个id的标签来表示当前数据的身份id，它的用法跟result标签相同。

当查询到相同id相同的数据时，juice会认为它们属于关联相同的数据。

既然是一对多，那么"多"的数据肯定是一个集合，我们这里用切片来表示。如上所示，Hobbies字段表示的是一个Hobby结构体的切片。

在xml中使用collection来表示集合的映射关系，它的用法跟association标签相同，只不过是collection的property指向的是
一个切片的字段。

.. attention::

    当使用collection标签时，它的同级标签中必须存在一个id标签，不然juice找不到哪些数据时相关联的。


自增主键映射
------------

当我们往数据库中插入一条数据时，如果这条数据的主键是自动生成的，那么我们需要获取这条数据的主键值，将这个值赋值给我们的结构体。

如果想要实现这个功能，我们需要需要满足以下条件:

1. 对应的sql.Driver需要支持 ``LastInsertId`` 方法。
2. 传入的参数必须是一个指向结构体的指针。
3. 在xml中执行的sql语句必须是 ``insert`` 语句。
4. 在xml中执行的insert语句的中必须指定 ``useGeneratedKeys`` 属性为 ``true`` 。
5. 在xml中执行的insert语句的中指定 ``keyProperty`` 属性为我们传入结构体主键字段名。当这个字段为空时，juice会自动从当前传入的结构体字段中寻找tag为 `autoincr:"true"` 的字段，如果找到了，那么就将这个字段作为主键字段。
6. 结构体主键字段的类型必须是可被 SetInt 的类型，如int64, int32, int等。

下面是一个例子:

.. code-block:: xml

    <mapper namespace="main">
        <insert id="InsertUser" useGeneratedKeys="true" keyProperty="Id">
            insert into user(name, age) values(#{name}, #{age})
        </insert>
    </mapper>

.. code-block:: go

    type User struct {
        Id   int64  `column:"id" autoincr:"true"`
        Name string `column:"name"`
        Age  int    `column:"age"`
    }

    user := User{
        Name: "张三",
        Age:  18,
    }
    _, err := engine.Object("main.InsertUser").Exec(&user)
    if err != nil {
        panic(err)
    }
    fmt.Println(user.Id)
