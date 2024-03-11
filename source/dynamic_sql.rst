动态sql
============

.. note::

    动态SQL是指在运行时（而不是编译时）根据不同条件动态地生成SQL语句的技术。通常，这些条件包括用户输入、程序逻辑或查询参数等。动态SQL可以帮助开发人员更加灵活地构建数据库查询，从而实现更好的性能和可维护性。在动态SQL中，应用程序会使用字符串拼接或参数化查询等技术来构建SQL查询语句。这些查询语句可能包含一个或多个查询参数，在运行时由应用程序提供。通过将查询参数与SQL查询语句结合使用，可以生成具有不同条件的不同查询语句。这些查询语句可以执行查询、插入、更新或删除等操作。动态SQL的优点是可以根据不同条件动态地生成查询语句，从而使查询更加灵活和可定制。然而，使用动态SQL也存在一些风险，例如SQL注入攻击等安全问题，因此需要谨慎使用。


juice 提供了动态sql的支持，可以通过动态sql来实现动态的sql语句。

动态sql的语法和xml语法类似，但是动态sql的语法更加简单，更加灵活。

动态sql的语法主要包括 ``if、where、set、foreach、trim、choose、when、otherwise`` 等。下面我们来看看这些语法的使用方法。

if
----

if语句用来判断条件，如果条件成立，则执行if语句中的sql语句，否则不执行。if语句的语法如下：

.. code-block:: xml

    <if test="条件表达式 // 如: 1 + 1 > 0">sql语句</if>

.. note::
    其中，条件表达式可以是任意的表达式，表达式运行结果值必须是boolean类型。如果条件表达式结果为true，则执行if语句中的sql语句，否则不执行。

下面我们来看一个例子：

.. code-block:: xml

    <select id="SelectByCondition">
        select * from user
        <if test='name != ""'>
            where name = #{name}
        </if>
    </select>

上面的例子中，如果name的值不为空，则在sql语句中添加 ``where name = #{name}``，否则不添加。如果name的值为空，则生成的sql语句为 ``select * from user``。

where
--------

where语句用来判断条件，如果条件成立，则执行where语句中的sql语句，否则不执行。where语句的语法如下

.. code-block:: xml

    <where>
        <if test="条件表达式">sql语句</if>
        <if test="条件表达式">sql语句</if>
        ...
    </where>

.. note::

    where语句中可以包含多个if语句，如果where语句中的所有if语句的条件表达式的值都为false，则不执行where语句中的sql语句。


如果where语句中的if语句的条件表达式的值都为true，则执行where语句中的sql语句，并且去除多余的and 和 or。下面我们来看一个例子：

.. code-block:: xml

    <select id="SelectByCondition">
        select * from user
        <where>
            <if test='name != ""'>
                and name = #{name}
            </if>
            <if test='age != 0'>
                and age = #{age}
            </if>
        </where>
    </select>

上面的例子中，如果name的值不为空，则在sql语句中添加 ``where name = #{name}``，否则不添加。

如果age的值不为0，则在sql语句中添加 ``where age = #{age}``，否则不添加。

如果name的值为空，age的值为0，则生成的sql语句为 ``select * from user``。

如果name的值不为空，age的值不为0，则生成的sql语句为 ``select * from user where name = #{name} and age = #{age}``。

set
-----

set语法用于更新语句中，用来设置更新的字段，它会在set语句中去除多余的逗号。set语法的语法如下：

.. code-block:: xml

    <set>
        <if test="条件表达式">sql语句</if>
        <if test="条件表达式">sql语句</if>
        ...
    </set>

.. note::
    set语句中可以包含多个if语句，如果set语句中的所有if语句的条件表达式的值都为false，则不执行set语句中的sql语句。如果set语句中的所有if语句的条件表达式的值都为true，则执行set语句中的sql语句。如果set语句中的if语句的条件表达式的值不全为true或false，则执行set语句中的sql语句，并且去除多余的逗号。


下面我们来看一个例子：

.. code-block:: xml

    <update id="UpdateByCondition">
        update user
        <set>
            <if test='name != ""'>
                name = #{name},
            </if>
            <if test='age != 0'>
                age = #{age},
            </if>
        </set>
        where id = #{id}
    </update>


上面的例子中，如果name的值不为空，则在sql语句中添加 ``name = #{name}``，否则不添加。sql 语句为 ``update user SET name = #{name} where id = #{id}``。

如果age的值不为0，则在sql语句中添加 ``age = #{age}``，否则不添加。sql 语句为 ``update user SET age = #{age} where id = #{id}``。

如果name的值为空，age的值为0，则生成的sql语句为 ``update user where id = #{id}``。错误的sql语句。

如果name的值不为空，age的值不为0，则生成的sql语句为 ``update user SET name = #{name}, age = #{age} where id = #{id}``。

foreach
----------

foreach语句用来遍历集合，将集合中的元素作为参数传递给sql语句。foreach语句的语法如下：

.. code-block:: xml

    <foreach collection="集合" item="元素" index="索引" open="开始符" close="结束符" separator="分隔符">
        sql语句
    </foreach>

.. note::
    其中，collection属性用来指定集合，item属性用来指定集合中的元素，index属性用来指定集合中的索引，open属性用来指定开始符，close属性用来指定结束符，separator属性用来指定分隔符。


当传递的 collection 是一个切片或者数组时，index 属性为元素在切片或者数组中的索引，item 属性为元素的值。当传递的 collection 是一个 map 时，index 属性为 map 中的 key，item 属性为 map 中的 value。


下面我们来看一个例子：

.. code-block:: xml

    <select id="SelectByIds">
        select * from user where id in
        <foreach collection="ids" item="id" open="(" close=")" separator=",">
            #{id}
        </foreach>
    </select>

上面的例子中，将ids集合中的元素作为参数传递给sql语句，生成的sql语句为 ``select * from user where id in (?, ?, ?)``。 ``?`` 为占位符（不同的驱动占位符不同），实际的值为 ``ids`` 集合中的元素。


trim
-------

trim语句用来去除sql语句中开头和结尾的多余的关键字，例如and和or。trim语句的语法如下：

.. code-block:: xml

    <trim prefix="前缀" prefixOverrides="前缀覆盖" suffix="后缀" suffixOverrides="后缀覆盖">
        sql语句
    </trim>


.. note::
    其中，prefix属性用来设置要添加到sql语句开头的关键字，suffix属性用来设置要添加到sql语句结尾的关键字，prefixOverrides属性用来设置要去除的前缀关键字列表，suffixOverrides属性用来设置要去除的后缀关键字列表。如果prefix属性和suffix属性都不设置，则不添加前缀和后缀关键字；如果prefixOverrides属性和suffixOverrides属性都不设置，则不去除前缀和后缀关键字。


下面我们来看一个例子：


.. code-block:: xml

    <select id="SelectByCondition">
        select * from user
        <trim prefix="where" prefixOverrides="and | or">
            <if test='name != ""'>
                and name = #{name}
            </if>
            <if test='age != 0'>
                and age = #{age}
            </if>
        </trim>
    </select>


上面的例子中，如果name的值不为空，则在sql语句中添加 ``where name = #{name}``，否则不添加。如果age的值不为0，则在sql语句中添加 ``where age = #{age}``，否则不添加。如果name的值为空，age的值为0，则生成的sql语句为 ``select * from user``。如果name的值不为空，age的值不为0，则生成的sql语句为``select * from user where name = #{name} and age = #{age}``。

choose、when和otherwise
----------------------------

choose、when、otherwise语句用来实现类似于switch语句的功能。choose语句相当于switch语句，when语句相当于case语句，otherwise语句相当于default语句。choose、when、otherwise语句的语法如下：

.. code-block:: xml

    <choose>
        <when test="条件表达式">sql语句</when>
        <when test="条件表达式">sql语句</when>
        ...
        <otherwise>sql语句</otherwise>
    </choose>


其中，when语句用来实现if语句的效果，otherwise语句用来实现else语句的效果。下面我们来看一个例子：

.. code-block:: xml

    <select id="SelectByCondition">
        select * from user
        <where>
            <choose>
                <when test='name != ""'>
                    and name = #{name}
                </when>
                <when test='age != 0'>
                    and age = #{age}
                </when>
                <otherwise>
                    and name = #{name} and age = #{age}
                </otherwise>
            </choose>
        </where>
    </select>


上面的例子中，如果name的值不为空，则在sql语句中添加 ``and name = #{name}``，否则不添加。如果age的值不为0，则在sql语句中添加 ``and age = #{age}``，否则不添加。如果name的值为空，age的值为0，则生成的sql语句为 ``select * from user where name = #{name} and age = #{age}``。如果name的值不为空，age的值不为0，则生成的sql语句为 ``select * from user where and name = #{name} and age = #{age}``。


otherwise语句用来实现else语句的效果，otherwise语句的语法如下：

.. code-block:: xml

    <otherwise>
        sql语句
    </otherwise>

当choose语句中的所有when语句的条件表达式都不成立时，执行otherwise语句。

sql、include
---------------

sql语句用来定义sql片段，include语句用来引用sql片段。

sql语句的语法如下：

.. code-block:: xml

    <mapper namespace="com.example.mapper.UserMapper">
        <sql id="columns">
            id, name, age
        </sql>

sql标签必须写在mapper标签中，id属性用来指定sql片段的id，sql片段的id属性在其所在的mapper中必须唯一。

include语句的语法如下：

.. code-block:: xml

    <select id="SelectAll">
        select
        <include refid="columns"/>
        from user
    </select>

include标签必须写在action标签中，refid属性用来指定要引用的sql片段的id。

如果include和sql标签都在同一个mapper中，则可以直接使用sql片段的id。

如果include和sql标签不在同一个mapper中，则需要使用namespace属性来指定sql片段所在的mapper的namespace，例如：

.. code-block:: xml

    <select id="SelectAll">
        select
        <include refid="com.example.mapper.UserMapper.columns"/>
        from user
    </select>


values、value
--------

values语句用来将参数值作为列值插入到表中。values只会在insert action中使用，values语句的语法如下：

.. code-block:: xml

    <insert id="Insert">
        insert into user
        <values>
        </values>
    </insert>

光有values语句是不够的，还需要使用value语句来指定参数值。value语句的语法如下：

.. code-block:: xml

    <insert id="Insert">
        insert into user
        <values>
            <value column="uid" value="#{uid}"/>
            <value column="create_at" value="NOW()"/>
            <value column="name"/>
        </values>
     </insert>

其中，column属性用来指定列名，value属性用来指定参数值。

注意：如果value属性不设置，则默认使用参数名作为占位符拼接到sql语句中，例如上面的例子中的name列，生成的sql语句为 ``insert into user (uid, create_at, name) values (#{uid}, NOW(), #{name})``。

同时，如果value属性的值为一个字符串字面值，则会直接将字符串字面值拼接到sql语句中，例如上面的例子中的create_at列，生成的sql语句为 ``insert into user (uid, create_at, name) values (#{uid}, NOW(), name)``。

alias、field
--------------

alias语句用来设置表的别名，field语句用来设置列的别名。alias 只在select action中使用，filed 是alias的子元素。alias和field语句的语法如下：

.. code-block:: xml

    <select id="Select">
        select
        <alias>
            <field column="uid" alias="id"/>
            <field column="name"/>
        </alias>
    </select>

其中，column属性用来指定列名，alias属性用来指定列的别名。当alias属性不设置时，使用column属性的值作为列的别名。

如上面的例子中，生成的sql语句为 ``select uid as id, name from user``。




