依赖注入与服务定位器（Dependency Injection/Service Location）
*************************************
The following example is a bit lengthy, but explains why use service location and dependency injection.
First, let's pretend we are developing a component called SomeComponent. This performs a task that is not important now.
Our component has some dependency that is a connection to a database.
下面的例子会有点长，但是很好的解释了我们为什么使用服务定位和依赖注入。
首先，假设我们正在开发的组件叫SomeComponent。
组件具体做什么业务不用关心，只需要知道这个组件需要连接数据库。

In this first example, the connection is created inside the component. This approach is impractical; due to the fact
we cannot change the connection parameters or the type of database system because the component only works as created.
在这个例子中，组件自己来创建数据库连接，但其实这种方法并不实用，
因为这样的话组件只能建立不能改变连接参数和数据库类型的连接。

.. code-block:: php

    <?php

    class SomeComponent
    {

        /**
         * The instantiation of the connection is hardcoded inside
         * the component, therefore it's difficult replace it externally
         * or change its behavior
         * 连接信息被硬编码到组件中，因此很难从外部对其进行更换和修改。
         */
        public function someDbTask()
        {
            $connection = new Connection(array(
                "host" => "localhost",
                "username" => "root",
                "password" => "secret",
                "dbname" => "invo"
            ));

            // ...
        }

    }

    $some = new SomeComponent();
    $some->someDbTask();

To solve this, we have created a setter that injects the dependency externally before using it. For now, this seems to be
a good solution:
为了解决这个问题，我们在组件中增加一个设置连接的方法，并在使用组件的时候才从外部传入数据库连接信息。就目前而言，这看起来是个很好的解决方案：

.. code-block:: php

    <?php

    class SomeComponent
    {

        protected $_connection;

        /**
         * Sets the connection externally
         * 设置数据库连接
         */
        public function setConnection($connection)
        {
            $this->_connection = $connection;
        }

        public function someDbTask()
        {
            $connection = $this->_connection;

            // ...
        }

    }

    $some = new SomeComponent();

    //Create the connection
    //建立连接
    $connection = new Connection(array(
        "host" => "localhost",
        "username" => "root",
        "password" => "secret",
        "dbname" => "invo"
    ));

    //Inject the connection in the component
    //将数据库连接注入到我们的组件中
    $some->setConnection($connection);

    $some->someDbTask();

Now consider that we use this component in different parts of the application and
then we will need to create the connection several times before passing it to the component.
Using some kind of global registry where we obtain the connection instance and not have
to create it again and again could solve this:
现在想一下，我们在一个网站应用的不同的地方要多次调用这个组件，因为每次调用都要建立一个数据库连接，那就需要建立很多个连接。
使用类似全局注册的方式，我们建立一个连接实例后，就不需要重复建立连接。顺着这个思路继续：

.. code-block:: php

    <?php

    class Registry
    {

        /**
         * Returns the connection
         * 全局注册类，返回数据库连接实例
         */
        public static function getConnection()
        {
           return new Connection(array(
                "host" => "localhost",
                "username" => "root",
                "password" => "secret",
                "dbname" => "invo"
            ));
        }

    }

    class SomeComponent
    {

        protected $_connection;

        /**
         * Sets the connection externally
         * 设置数据库连接
         */
        public function setConnection($connection)
        {
            $this->_connection = $connection;
        }

        public function someDbTask()
        {
            $connection = $this->_connection;

            // ...
        }

    }

    $some = new SomeComponent();

    //Pass the connection defined in the registry
    //使用全局注册的连接实例
    $some->setConnection(Registry::getConnection());

    $some->someDbTask();

Now, let's imagine that we must implement two methods in the component, the first always need to create a new connection and the second always need to use a shared connection:
现在，我们再想一下，我们必须给组件实现两个方法，第一种是总是创建新的数据库连接，第二种使用共享连接。

.. code-block:: php

    <?php

    class Registry
    {

        protected static $_connection;

        /**
         * Creates a connection
         * 建立连接
         */
        protected static function _createConnection()
        {
            return new Connection(array(
                "host" => "localhost",
                "username" => "root",
                "password" => "secret",
                "dbname" => "invo"
            ));
        }

        /**
         * Creates a connection only once and returns it
         * 如果有连接就直接返回，没有的话重新建立
         */
        public static function getSharedConnection()
        {
            if (self::$_connection===null){
                $connection = self::_createConnection();
                self::$_connection = $connection;
            }
            return self::$_connection;
        }

        /**
         * Always returns a new connection
         * 一直返回新的连接
         */
        public static function getNewConnection()
        {
            return self::_createConnection();
        }

    }

    class SomeComponent
    {

        protected $_connection;

        /**
         * Sets the connection externally
         * 设置数据库连接
         */
        public function setConnection($connection)
        {
            $this->_connection = $connection;
        }

        /**
         * This method always needs the shared connection
         * 这个方法一直使用共享连接
         */
        public function someDbTask()
        {
            $connection = $this->_connection;

            // ...
        }

        /**
         * This method always needs a new connection
         * 这个方法则一直使用新连接
         */
        public function someOtherDbTask($connection)
        {

        }

    }

    $some = new SomeComponent();

    //This injects the shared connection
    //这里将使用共享连接
    $some->setConnection(Registry::getSharedConnection());

    $some->someDbTask();

    //Here, we always pass a new connection as parameter
    //而这里，我们需要一直传一个新连接做参数
    $some->someOtherDbTask(Registry::getConnection());

So far we have seen how dependency injection solved our problems. Passing dependencies as arguments instead
of creating them internally in the code makes our application more maintainable and decoupled. However, in the long-term,
this form of dependency injection have some disadvantages.
到目前为止，我们已经看到依赖注入解决了我们的问题。将依赖作为参数传递而不是在代码内部建立它，使得我们的应用更易于维护和解耦，但是从长远来看，
依赖注入又有一些缺点。

For instance, if the component has many dependencies, we will need to create multiple setter arguments to pass
the dependencies or create a constructor that pass them with many arguments, additionally creating dependencies
before using the component, every time, makes our code not as maintainable as we would like:
例如，如果该组件有很多的依赖，我们就需要为每一个依赖创建一个赋值方法来进行参数传递或者创建一个可以传递多个参数的构造器，另外,
每一次使用该组件之前都要创建依赖，让我们代码可维护性并没有达到预期：
.. code-block:: php

    <?php

    //Create the dependencies or retrieve them from the registry
    //建立依赖或者从注册方法里搜索他们
    $connection = new Connection();
    $session = new Session();
    $fileSystem = new FileSystem();
    $filter = new Filter();
    $selector = new Selector();

    //Pass them as constructor parameters
    //作为构造方法的参数传递
    $some = new SomeComponent($connection, $session, $fileSystem, $filter, $selector);

    // ... or using setters
    // ... 或者使用赋值方法

    $some->setConnection($connection);
    $some->setSession($session);
    $some->setFileSystem($fileSystem);
    $some->setFilter($filter);
    $some->setSelector($selector);

Think we had to create this object in many parts of our application. If you ever do not require any of the dependencies,
we need to go everywhere to remove the parameter in the constructor or the setter where we injected the code. To solve this,
we return again to a global registry to create the component. However, it adds a new layer of abstraction before creating
the object:
想一想，我们需要在应用的许多地方创建此对象。一旦不再需要某个依赖，我们还需要在任何使用该组件的地方重复删除相关构造方法或赋值方法的参数。
为了解决这个问题，我们重新回到建立该组件的全局注册表，并且在创建对象前增加一个新的抽象层：

.. code-block:: php

    <?php

    class SomeComponent
    {

        // ...

        /**
         * Define a factory method to create SomeComponent instances injecting its dependencies
         * 定义一个注入依赖的工厂方法来创建该组件的实例
         */
        public static function factory()
        {

            $connection = new Connection();
            $session = new Session();
            $fileSystem = new FileSystem();
            $filter = new Filter();
            $selector = new Selector();

            return new self($connection, $session, $fileSystem, $filter, $selector);
        }

    }

One moment, we returned to the beginning, we are again building the dependencies inside of the component! We can move on and find out a way
to solve this problem every time. But it seems that time and again we fall back into bad practices.
啊哦，我们又回到了最初，我们有把依赖硬编码到组件内部！每一次我们都能找到一个解决问题的办法。但是一次又一次的我们再次失败。

A practical and elegant way to solve these problems is using a container for dependencies. The containers act as the global registry that
we saw earlier. Using the container for dependencies as a bridge to obtain the dependencies allows us to reduce the complexity
of our component:
使用依赖容器是解决这些问题最实际和优雅的方法。我们前面看到过，容器可以作为全局注册表。使用依赖容器作为桥梁来获取依赖可以让我们的组件降低复杂度。

.. code-block:: php

    <?php

    class SomeComponent
    {

        protected $_di;

        public function __construct($di)
        {
            $this->_di = $di;
        }

        public function someDbTask()
        {

            // Get the connection service
            // Always returns a new connection
            // 获取连接服务
            // 一直返回一个新连接
            $connection = $this->_di->get('db');

        }

        public function someOtherDbTask()
        {

            // Get a shared connection service,
            // this will return the same connection everytime
            // 获取一个共享连接服务
            // 每次都返回同样的连接
            $connection = $this->_di->getShared('db');

            //This method also requires an input filtering service
            //这个方法还需要一个输入过滤服务
            $filter = $this->_di->get('filter');

        }

    }

    $di = new Phalcon\DI();

    //Register a "db" service in the container
    //注册一个‘db’服务到容器
    $di->set('db', function() {
        return new Connection(array(
            "host" => "localhost",
            "username" => "root",
            "password" => "secret",
            "dbname" => "invo"
        ));
    });

    //Register a "filter" service in the container
    //注册一个‘filter’服务到容器
    $di->set('filter', function() {
        return new Filter();
    });

    //Register a "session" service in the container
    //注册一个‘session’服务到容器
    $di->set('session', function() {
        return new Session();
    });

    //Pass the service container as unique parameter
    //将该服务容器作为唯一的参数传递给组件
    $some = new SomeComponent($di);

    $some->someTask();

The component now simply access the service it requires when it needs it, if it does not require a service that is not even initialized
saving resources. The component is now highly decoupled. For example, we can replace the manner in which connections are created,
their behavior or any other aspect of them and that would not affect the component.
现在组件只在需要的时候才会访问服务，如果不需要某个服务，该服务甚至不会初始化，从而节省资源。该组件现在高度解耦。例如我们在每次创建连接前替换服务，
他们的行为或者任何其它方面都不会影响到组件的使用。

实现方法（Our approach）
============
Phalcon\\DI is a component implementing Dependency Injection and Location of services and it's itself a container for them.
Phalcon\\DI 是一个实现了依赖注入和位置服务的组件，它本身还是一个服务容器。

Since Phalcon is highly decoupled, Phalcon\\DI is essential to integrate the different components of the framework. The developer can
also use this component to inject dependencies and manage global instances of the different classes used in the application.
由于 Phalcon 高度解耦，Phalcon\\DI 是整合框架中不同组件必不可少的一部分。开发者还可以在应用中使用这个组件来注入依赖关系从而管理不同类的全局实例。

Basically, this component implements the `Inversion of Control`_ pattern. Applying this, the objects do not receive their dependencies
using setters or constructors, but requesting a service dependency injector. This reduces the overall complexity since there is only
one way to get the required dependencies within a component.
基本上，这个组件实现了控制模式反转。基于此，对象不在需要用赋值方法或者构造方法来处理依赖关系。
因为只有这一种方式来获取某个组件的所有需要的依赖，从而大大降低了整体的复杂度。

Additionally, this pattern increases testability in the code, thus making it less prone to errors.
此外，这种方式还增加了代码的可测试性，从而使代码更不容易出现错误。

使用容器注册服务（Registering services in the Container）
=====================================
The framework itself or the developer can register services. When a component A requires component B (or an instance of its class) to operate, it
can request component B from the container, rather than creating a new instance component B.
框架本身或者开发者都可以注册服务。当组件A的操作需要组件B（或者它的实例），组件A就可以从依赖容器中请求组件B，而不需要为组件B创建实例。

This way of working gives us many advantages:
这种工作方式为我们提供了很多好处：

* We can easily replace a component with one created by ourselves or a third party.
* 可以很容易的用自己的或者第三方的组件替换现在的。
* We have full control of the object initialization, allowing us to set these objects, as needed before delivering them to components.
* 拥有对象初始化的完整控制权，根据需要来设置组件需要对象。
* We can get global instances of components in a structured and unified way
* 可以在组件里用统一的方式获取全局实例

Services can be registered using several types of definitions:
定时服务的几种方式：

.. code-block:: php

    <?php

    //Create the Dependency Injector Container
    //创建依赖注入容器
    $di = new Phalcon\DI();

    //By its class name
    //使用类名称
    $di->set("request", 'Phalcon\Http\Request');

    //Using an anonymous function, the instance will be lazy loaded
    //使用匿名函数，实例会被延迟加载
    $di->set("request", function() {
        return new Phalcon\Http\Request();
    });

    //Registering an instance directly
    //直接注册实例
    $di->set("request", new Phalcon\Http\Request());

    //Using an array definition
    //数组方式定义
    $di->set("request", array(
        "className" => 'Phalcon\Http\Request'
    ));

The array syntax is also allowed to register services:
注册服务还可以使用数组语法方式

.. code-block:: php

    <?php

    //Create the Dependency Injector Container
    //创建依赖注入容器
    $di = new Phalcon\DI();

    //By its class name
    //使用类名称
    $di["request"] = 'Phalcon\Http\Request';

    //Using an anonymous function, the instance will be lazy loaded
    //使用匿名函数，实例会被延迟加载
    $di["request"] = function() {
        return new Phalcon\Http\Request();
    };

    //Registering an instance directly
    //直接注册实例
    $di["request"] = new Phalcon\Http\Request();

    //Using an array definition
    //数组方式定义
    $di["request"] = array(
        "className" => 'Phalcon\Http\Request'
    );

In the examples above, when the framework needs to access the request data, it will ask for the service identified as ‘request’ in the container.
The container in turn will return an instance of the required service. A developer might eventually replace a component when he/she needs.
在上面的例子中，当框架需要访问请求数据，它将在容器中查找并定位到‘request’服务。容器将返回一个请求服务的实例。只要开发者需要，就可以替换掉这个组件。

Each of the methods (demonstrated in the examples above) used to set/register a service has advantages and disadvantages. It is up to the
developer and the particular requirements that will designate which one is used.
上面例子中的每一种设置或注册服务的方法都有它的优点和缺点。具体用哪一种，取决于开发者的习惯和特殊的需求。

Setting a service by a string is simple, but lacks flexibility. Setting services using an array offers a lot more flexibility, but makes the
code more complicated. The lambda function is a good balance between the two, but could lead to more maintenance than one would expect.
用字符串方式设置服务是非常简单的，但是缺乏一些弹性。用数组设置服务则提供了更多的灵活性，但会使代码更复杂。匿名函数在这两者之间找到了一个很好的平衡，但是可能会带来一些维护难度。

Phalcon\\DI offers lazy loading for every service it stores. Unless the developer chooses to instantiate an object directly and store it
in the container, any object stored in it (via array, string, etc.) will be lazy loaded i.e. instantiated only when requested.
Phalcon\\DI 可以为保存其中的任何服务提供延迟加载，除非开发者选择将实例直接保存进容器。保存在容器中的任何对象（通过数组，字符串等方式）都会在它被请求的时候才延迟加载或者说实例化。

简单的注册（Simple Registration）
-------------------
As seen before, there are several ways to register services. These we call simple:
如上所叙，注册服务的方式是多重多样的。这些方式我们称之为简单方式。

String（字符串）
^^^^^^
This type expects the name of a valid class, returning an object of the specified class, if the class is not loaded it will be instantiated using an auto-loader.
This type of definition does not allow to specify arguments for the class constructor or parameters:
这种类型需要使用有效的类的名称，返回指定的类的对象，如果类还没有加载，它将被自动加载机制自动加载并实例化。
这种定义类型不允许为类的构造方法传递指定的参数。

.. code-block:: php

    <?php

    // return new Phalcon\Http\Request();
    // 返回 new Phalcon\Http\Request()
    $di->set('request', 'Phalcon\Http\Request');

对象（Object）
^^^^^^
This type expects an object. Due to the fact that object does not need to be resolved as it is
already an object, one could say that it is not really a dependency injection,
however it is useful if you want to force the returned dependency to always be
the same object/value:
这种方式需要一个对象。由于它已经是一个对象而不需要处理，你可以说这种方式不是真正的依赖注入，
然而如果你想强制它一直返回同样的对象/值的时候，这种方式是很有用的。

.. code-block:: php

    <?php

    // return new Phalcon\Http\Request();
    // 返回 new Phalcon\Http\Request()
    $di->set('request', new Phalcon\Http\Request());

闭包与匿名函数（Closures/Anonymous functions）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This method offers greater freedom to build the dependency as desired, however, it is difficult to
change some of the parameters externally without having to completely change the definition of dependency:
如同希望的一样，这种方式为建立依赖提供了更大的自由，然而，修改不需要完全修改的依赖定义的外部参数是困难的（好绕口的一句话）。

.. code-block:: php

    <?php

    $di->set("db", function() {
        return new \Phalcon\Db\Adapter\Pdo\Mysql(array(
             "host" => "localhost",
             "username" => "root",
             "password" => "secret",
             "dbname" => "blog"
        ));
    });

Some of the limitations can be overcome by passing additional variables to the closure's environment:
一些不方便的地方可以通过传递附加变量到封闭函数来解决

.. code-block:: php

    <?php

    //Using the $config variable in the current scope
    //在当前范围内使用变量 $config
    $di->set("db", function() use ($config) {
        return new \Phalcon\Db\Adapter\Pdo\Mysql(array(
             "host" => $config->host,
             "username" => $config->username,
             "password" => $config->password,
             "dbname" => $config->name
        ));
    });

复杂的注册（Complex Registration）
--------------------
If it is required to change the definition of a service without instantiating/resolving the service,
then, we need to define the services using the array syntax. Define a service using an array definition
can be a little more verbose:

.. code-block:: php

    <?php

    //Register a service 'logger' with a class name and its parameters
    $di->set('logger', array(
        'className' => 'Phalcon\Logger\Adapter\File',
        'arguments' => array(
            array(
                'type' => 'parameter',
                'value' => '../apps/logs/error.log'
            )
        )
    ));

    //Using an anonymous function
    $di->set('logger', function() {
        return new \Phalcon\Logger\Adapter\File('../apps/logs/error.log');
    });

Both service registrations above produce the same result. The array definition however, allows for alteration of the service parameters if needed:

.. code-block:: php

    <?php

    //Change the service class name
    $di->getService('logger')->setClassName('MyCustomLogger');

    //Change the first parameter without instantiating the logger
    $di->getService('logger')->setParameter(0, array(
        'type' => 'parameter',
        'value' => '../apps/logs/error.log'
    ));

In addition by using the array syntax you can use three types of dependency injection:

构造函数注入（Constructor Injection）
^^^^^^^^^^^^^^^^^^^^^
This injection type passes the dependencies/arguments to the class constructor.
Let's pretend we have the following component:

.. code-block:: php

    <?php

    namespace SomeApp;

    use Phalcon\Http\Response;

    class SomeComponent
    {

        protected $_response;

        protected $_someFlag;

        public function __construct(Response $response, $someFlag)
        {
            $this->_response = $response;
            $this->_someFlag = $someFlag;
        }

    }

The service can be registered this way:

.. code-block:: php

    <?php

    $di->set('response', array(
        'className' => 'Phalcon\Http\Response'
    ));

    $di->set('someComponent', array(
        'className' => 'SomeApp\SomeComponent',
        'arguments' => array(
            array('type' => 'service', 'name' => 'response'),
            array('type' => 'parameter', 'value' => true)
        )
    ));

The service "response" (Phalcon\\Http\\Response) is resolved to be passed as the first argument of the constructor,
while the second is a boolean value (true) that is passed as it is.

设值注入（Setter Injection）
^^^^^^^^^^^^^^^^
Classes may have setters to inject optional dependencies, our previous class can be changed to accept the dependencies with setters:

.. code-block:: php

    <?php

    namespace SomeApp;

    use Phalcon\Http\Response;

    class SomeComponent
    {

        protected $_response;

        protected $_someFlag;

        public function setResponse(Response $response)
        {
            $this->_response = $response;
        }

        public function setFlag($someFlag)
        {
            $this->_someFlag = $someFlag;
        }

    }

A service with setter injection can be registered as follows:

.. code-block:: php

    <?php

    $di->set('response', array(
        'className' => 'Phalcon\Http\Response'
    ));

    $di->set('someComponent', array(
        'className' => 'SomeApp\SomeComponent',
        'calls' => array(
            array(
                'method' => 'setResponse',
                'arguments' => array(
                    array('type' => 'service', 'name' => 'response'),
                )
            ),
            array(
                'method' => 'setFlag',
                'arguments' => array(
                    array('type' => 'parameter', 'value' => true)
                )
            )
        )
    ));

属性注入（Properties Injection）
^^^^^^^^^^^^^^^^^^^^
A less common strategy is to inject dependencies or parameters directly into public attributes of the class:

.. code-block:: php

    <?php

    namespace SomeApp;

    use Phalcon\Http\Response;

    class SomeComponent
    {

        public $response;

        public $someFlag;

    }

A service with properties injection can be registered as follows:

.. code-block:: php

    <?php

    $di->set('response', array(
        'className' => 'Phalcon\Http\Response'
    ));

    $di->set('someComponent', array(
        'className' => 'SomeApp\SomeComponent',
        'properties' => array(
            array(
                'name' => 'response',
                'value' => array('type' => 'service', 'name' => 'response')
            ),
            array(
                'name' => 'someFlag',
                'value' => array('type' => 'parameter', 'value' => true)
            )
        )
    ));

Supported parameter types include the following:

+-------------+----------------------------------------------------------+-------------------------------------------------------------------------------------+
| Type        | Description                                              | Example                                                                             |
+=============+==========================================================+=====================================================================================+
| parameter   | Represents a literal value to be passed as parameter     | array('type' => 'parameter', 'value' => 1234)                                       |
+-------------+----------------------------------------------------------+-------------------------------------------------------------------------------------+
| service     | Represents another service in the service container      | array('type' => 'service', 'name' => 'request')                                     |
+-------------+----------------------------------------------------------+-------------------------------------------------------------------------------------+
| instance    | Represents an object that must be built dynamically      | array('type' => 'instance', 'className' => 'DateTime', 'arguments' => array('now')) |
+-------------+----------------------------------------------------------+-------------------------------------------------------------------------------------+

Resolving a service whose definition is complex may be slightly slower than simple definitions seen previously. However,
these provide a more robust approach to define and inject services.

Mixing different types of definitions is allowed, everyone can decide what is the most appropriate way to register the services
according to the application needs.

服务解疑（Resolving Services）
==================
Obtaining a service from the container is a matter of simply calling the “get” method. A new instance of the service will be returned:

.. code-block:: php

    <?php $request = $di->get("request");

Or by calling through the magic method:

.. code-block:: php

    <?php

    $request = $di->getRequest();

Or using the array-access syntax:

.. code-block:: php

    <?php

    $request = $di['request'];

Arguments can be passed to the constructor by adding an array parameter to the method "get":

.. code-block:: php

    <?php

    // new MyComponent("some-parameter", "other")
    $component = $di->get("MyComponent", array("some-parameter", "other"));

共享服务（Shared services）
===============
Services can be registered as "shared" services this means that they always will act as singletons_. Once the service is resolved for the first time
the same instance of it is returned every time a consumer retrieve the service from the container:

.. code-block:: php

    <?php

    //Register the session service as "always shared"
    $di->setShared('session', function() {
        $session = new Phalcon\Session\Adapter\Files();
        $session->start();
        return $session;
    });

    $session = $di->get('session'); // Locates the service for the first time
    $session = $di->getSession(); // Returns the first instantiated object

An alternative way to register shared services is to pass "true" as third parameter of "set":

.. code-block:: php

    <?php

    //Register the session service as "always shared"
    $di->set('session', function() {
        //...
    }, true);

If a service isn't registered as shared and you want to be sure that a shared instance will be accessed every time
the service is obtained from the DI, you can use the 'getShared' method:

.. code-block:: php

    <?php

    $request = $di->getShared("request");

单独操作服务（Manipulating services individually）
==================================
Once a service is registered in the service container, you can retrieve it to manipulate it individually:

.. code-block:: php

    <?php

    //Register the "register" service
    $di->set('request', 'Phalcon\Http\Request');

    //Get the service
    $requestService = $di->getService('request');

    //Change its definition
    $requestService->setDefinition(function() {
        return new Phalcon\Http\Request();
    });

    //Change it to shared
    $requestService->setShared(true);

    //Resolve the service (return a Phalcon\Http\Request instance)
    $request = $requestService->resolve();

通过服务容器实例化类（Instantiating classes via the Service Container）
===============================================
When you request a service to the service container, if it can't find out a service with the same name it'll try to load a class with
the same name. With this behavior we can replace any class by another simply by registering a service with its name:

.. code-block:: php

    <?php

    //Register a controller as a service
    $di->set('IndexController', function() {
        $component = new Component();
        return $component;
    }, true);

    //Register a controller as a service
    $di->set('MyOtherComponent', function() {
        //Actually returns another component
        $component = new AnotherComponent();
        return $component;
    });

    //Create an instance via the service container
    $myComponent = $di->get('MyOtherComponent');

You can take advantage of this, always instantiating your classes via the service container (even if they aren't registered as services). The DI will
fallback to a valid autoloader to finally load the class. By doing this, you can easily replace any class in the future by implementing a definition
for it.

自动注入 DI（Automatic Injecting of the DI itself）
====================================
If a class or component requires the DI itself to locate services, the DI can automatically inject itself to the instances it creates,
to do this, you need to implement the :doc:`Phalcon\\DI\\InjectionAwareInterface <../api/Phalcon_DI_InjectionAwareInterface>` in your classes:

.. code-block:: php

    <?php

    class MyClass implements \Phalcon\DI\InjectionAwareInterface
    {

        protected $_di;

        public function setDi($di)
        {
            $this->_di = $di;
        }

        public function getDi()
        {
            return $this->_di;
        }

    }

Then once the service is resolved, the $di will be passed to setDi automatically:

.. code-block:: php

    <?php

    //Register the service
    $di->set('myClass', 'MyClass');

    //Resolve the service (NOTE: $myClass->setDi($di) is automatically called)
    $myClass = $di->get('myClass');

避免服务解析（Avoiding service resolution）
===========================
Some services are used in each of the requests made to the application, eliminate the process of resolving the service
could add some small improvement in performance.

.. code-block:: php

    <?php

    //Resolve the object externally instead of using a definition for it:
    $router = new MyRouter();

    //Pass the resolved object to the service registration
    $di->set('router', $router);

使用文件组织服务（Organizing services in files）
============================
You can better organize your application by moving the service registration to individual files instead of
doing everything in the application's bootstrap:

.. code-block:: php

    <?php

    $di->set('router', function() {
        return include "../app/config/routes.php";
    });

Then in the file ("../app/config/routes.php") return the object resolved:

.. code-block:: php

    <?php

    $router = new MyRouter();

    $router->post('/login');

    return $router;

使用静态的方式访问注入器（Accessing the DI in a static way）
================================
If needed you can access the latest DI created in a static function in the following way:

.. code-block:: php

    <?php

    class SomeComponent
    {

        public static function someMethod()
        {
            //Get the session service
            $session = Phalcon\DI::getDefault()->getSession();
        }

    }

注入器默认工厂（Factory Default DI）
==================
Although the decoupled character of Phalcon offers us great freedom and flexibility, maybe we just simply want to use it as a full-stack
framework. To achieve this, the framework provides a variant of Phalcon\\DI called Phalcon\\DI\\FactoryDefault. This class automatically
registers the appropriate services bundled with the framework to act as full-stack.

.. code-block:: php

    <?php $di = new Phalcon\DI\FactoryDefault();

服务名称约定（Service Name Conventions）
========================
Although you can register services with the names you want, Phalcon has a several naming conventions that allow it to get the
the correct (built-in) service when you need it.

+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| Service Name        | Description                                 | Default                                                                                            | Shared |
+=====================+=============================================+====================================================================================================+========+
| dispatcher          | Controllers Dispatching Service             | :doc:`Phalcon\\Mvc\\Dispatcher <../api/Phalcon_Mvc_Dispatcher>`                                    | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| router              | Routing Service                             | :doc:`Phalcon\\Mvc\\Router <../api/Phalcon_Mvc_Router>`                                            | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| url                 | URL Generator Service                       | :doc:`Phalcon\\Mvc\\Url <../api/Phalcon_Mvc_Url>`                                                  | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| request             | HTTP Request Environment Service            | :doc:`Phalcon\\Http\\Request <../api/Phalcon_Http_Request>`                                        | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| response            | HTTP Response Environment Service           | :doc:`Phalcon\\Http\\Response <../api/Phalcon_Http_Response>`                                      | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| cookies             | HTTP Cookies Management Service             | :doc:`Phalcon\\Http\\Response\\Cookies <../api/Phalcon_Http_Response_Cookies>`                     | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| filter              | Input Filtering Service                     | :doc:`Phalcon\\Filter <../api/Phalcon_Filter>`                                                     | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| flash               | Flash Messaging Service                     | :doc:`Phalcon\\Flash\\Direct <../api/Phalcon_Flash_Direct>`                                        | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| flashSession        | Flash Session Messaging Service             | :doc:`Phalcon\\Flash\\Session <../api/Phalcon_Flash_Session>`                                      | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| session             | Session Service                             | :doc:`Phalcon\\Session\\Adapter\\Files <../api/Phalcon_Session_Adapter_Files>`                     | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| eventsManager       | Events Management Service                   | :doc:`Phalcon\\Events\\Manager <../api/Phalcon_Events_Manager>`                                    | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| db                  | Low-Level Database Connection Service       | :doc:`Phalcon\\Db <../api/Phalcon_Db>`                                                             | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| security            | Security helpers                            | :doc:`Phalcon\\Security <../api/Phalcon_Security>`                                                 | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| crypt               | Encrypt/Decrypt data                        | :doc:`Phalcon\\Crypt <../api/Phalcon_Crypt>`                                                       | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| tag                 | HTML generation helpers                     | :doc:`Phalcon\\Tag <../api/Phalcon_Tag>`                                                           | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| escaper             | Contextual Escaping                         | :doc:`Phalcon\\Escaper <../api/Phalcon_Escaper>`                                                   | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| annotations         | Annotations Parser                          | :doc:`Phalcon\\Annotations\\Adapter\\Memory <../api/Phalcon_Annotations_Adapter_Memory>`           | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| modelsManager       | Models Management Service                   | :doc:`Phalcon\\Mvc\\Model\\Manager <../api/Phalcon_Mvc_Model_Manager>`                             | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| modelsMetadata      | Models Meta-Data Service                    | :doc:`Phalcon\\Mvc\\Model\\MetaData\\Memory <../api/Phalcon_Mvc_Model_MetaData_Memory>`            | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| transactionManager  | Models Transaction Manager Service          | :doc:`Phalcon\\Mvc\\Model\\Transaction\\Manager <../api/Phalcon_Mvc_Model_Transaction_Manager>`    | Yes    |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| modelsCache         | Cache backend for models cache              | None                                                                                               | -      |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+
| viewsCache          | Cache backend for views fragments           | None                                                                                               | -      |
+---------------------+---------------------------------------------+----------------------------------------------------------------------------------------------------+--------+

自定义注入器（Implementing your own DI）
========================
The :doc:`Phalcon\\DiInterface <../api/Phalcon_DiInterface>` interface must be implemented to create your own DI replacing the one provided by Phalcon or extend the current one.

.. _`Inversion of Control`: http://en.wikipedia.org/wiki/Inversion_of_control
.. _Singletons: http://en.wikipedia.org/wiki/Singleton_pattern
