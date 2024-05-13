中间件
=========


什么是中间件
---------------

在juice中，中间件是一个接口，接口的描述如下：

.. code:: go

    // QueryHandler defines the handler of the query.
    type QueryHandler func(ctx context.Context, query string, args ...any) (*sql.Rows, error)

    // ExecHandler defines the handler of the exec.
    type ExecHandler func(ctx context.Context, query string, args ...any) (sql.Result, error)

    // Middleware is a wrapper of QueryHandler and ExecHandler.
    type Middleware interface {
    	// QueryContext wraps the QueryHandler.
    	QueryContext(stmt Statement, next QueryHandler) QueryHandler
    	// ExecContext wraps the ExecHandler.
    	ExecContext(stmt Statement, next ExecHandler) ExecHandler
    }

中间件的作用是在执行SQL语句之前，对SQL语句进行一些处理，比如SQL语句的日志记录，SQL语句的缓存等等。

当执行的是查询操作的时候，``QueryContext`` 将会被执行。否则执行 ``ExecContext``

juice中内置了一些中间件，比如：

- :class:`juice/middleware/DebugMiddleware`：用于打印SQL语句的中间件。

.. code:: go

    // logger is a default logger for debug.
    var logger = log.New(log.Writer(), "[juice] ", log.Flags())

    // DebugMiddleware is a middleware that prints the sql statement and the execution time.
    type DebugMiddleware struct{}

    // QueryContext implements Middleware.
    // QueryContext will print the sql statement and the execution time.
    func (m *DebugMiddleware) QueryContext(stmt Statement, next QueryHandler) QueryHandler {
    	if !m.isBugMode(stmt) {
    		return next
    	}
    	// wrapper QueryHandler
    	return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
    		start := time.Now()
    		rows, err := next(ctx, query, args...)
    		spent := time.Since(start)
    		logger.Printf("\x1b[33m[%s]\x1b[0m \x1b[32m %s\x1b[0m \x1b[34m %v\x1b[0m \x1b[31m %v\x1b[0m\n", stmt.Name(), query, args, spent)
    		return rows, err
    	}
    }

    // ExecContext implements Middleware.
    // ExecContext will print the sql statement and the execution time.
    func (m *DebugMiddleware) ExecContext(stmt Statement, next ExecHandler) ExecHandler {
    	if !m.isBugMode(stmt) {
    		return next
    	}
    	// wrapper ExecContext
    	return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
    		start := time.Now()
    		rows, err := next(ctx, query, args...)
    		spent := time.Since(start)
    		logger.Printf("\x1b[33m[%s]\x1b[0m \x1b[32m %s\x1b[0m \x1b[34m %v\x1b[0m \x1b[31m %v\x1b[0m\n", stmt.Name(), query, args, spent)
    		return rows, err
    	}
    }

    // isBugMode returns true if the debug mode is on.
    // Default debug mode is on.
    // You can turn off the debug mode by setting the debug tag to false in the mapper statement attribute or the configuration.
    func (m *DebugMiddleware) isBugMode(stmt Statement) bool {
    	// try to one the bug mode from the Statement
    	debug := stmt.Attribute("debug")
    	// if the bug mode is not set, try to one the bug mode from the Context
    	if debug == "false" {
    		return false
    	}
    	if cfg := stmt.Configuration(); cfg.Settings.Get("debug") == "false" {
    		return false
    	}
    	return true
    }

当你启用了这个中间件，juice会将每次执行的sql语句和参数写入到log包的默认的writer里面（默认是console），并且记录耗时。

当不想使用这个中间件的时候，可以在setting里面将debug设置为false, 这样就会全局关闭这个中间件。

.. code:: xml

    <settings>
        <setting name="debug" value="false"/>
    </settings>

当你想局部禁用这个功能的时候，可以在对应的action上面配置，如：

.. code:: xml

    <insert id="xxx" debug="false">
    </insert>


- :class:`juice/middleware/TimeoutMiddleware`：用于控制sql执行超时。

.. code-block:: go

    // TimeoutMiddleware is a middleware that sets the timeout for the sql statement.
    type TimeoutMiddleware struct{}

    // QueryContext implements Middleware.
    // QueryContext will set the timeout for the sql statement.
    func (t TimeoutMiddleware) QueryContext(stmt Statement, next QueryHandler) QueryHandler {
    	timeout := t.getTimeout(stmt)
    	if timeout <= 0 {
    		return next
    	}
    	return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
    		ctx, cancel := context.WithTimeout(ctx, time.Duration(timeout)*time.Millisecond)
    		defer cancel()
    		return next(ctx, query, args...)
    	}
    }

    // ExecContext implements Middleware.
    // ExecContext will set the timeout for the sql statement.
    func (t TimeoutMiddleware) ExecContext(stmt Statement, next ExecHandler) ExecHandler {
    	timeout := t.getTimeout(stmt)
    	if timeout <= 0 {
    		return next
    	}
    	return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
    		ctx, cancel := context.WithTimeout(ctx, time.Duration(timeout)*time.Millisecond)
    		defer cancel()
    		return next(ctx, query, args...)
    	}
    }

    // getTimeout returns the timeout from the Statement.
    func (t TimeoutMiddleware) getTimeout(stmt Statement) (timeout int64) {
    	timeoutAttr := stmt.Attribute("timeout")
    	if timeoutAttr == "" {
    		return
    	}
    	timeout, _ = strconv.ParseInt(timeoutAttr, 10, 64)
    	return
    }

在对应action标签的属性上面加上timeout属性，即可启用这个功能，timeout的单位为毫秒，如：

.. code-block:: xml

    <insert id="xxx" timeout="1000">
    </insert>

.. attention::

	注意：TimeoutMiddleware是在go语言级别实现的超时，而不是数据库级别。

自定义中间件
-------------

自定义中间只需要实现 ``Middleware`` 接口, 然后注册入对应的engine即可，如：

.. code-block:: go

    func main() {
        var mymiddleware Middleware = yourMiddlewareImpl{}

        cfg, err := juice.NewXMLConfiguration("config.xml")
        if err != nil {
            panic(err)
        }

        engine, err := juice.DefaultEngine(cfg)
        if err != nil {
            panic(err)
        }

        engine.Use(mymiddleware)
    }




