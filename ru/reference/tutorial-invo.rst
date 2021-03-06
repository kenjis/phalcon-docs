Урок 2: Приложение для создания счетов INVO
===========================
Во втором уроке мы создадим более сложное приложение с помощью Phalcon. INVO это одно из приложений, которое мы создали в качестве примера. INVO это небольшой сайт, который позволяет своим пользователям создавать счета и выполнять другие задачи для управления своими клиентами и продуктами. Полный код проекта можно клонировать из Github_.

INVO использует `Twitter Bootstrap`_ в качестве фронтенд-фреймворка. Кроме того, приложение не будет генерировать счета, оно служит для понимая того, как работает фреймворк.

Структура проекта
-----------------
После того как вы склонируете проект в корневой каталог вы увидите следующую структуру:

.. code-block:: bash

    invo/
        app/
            app/config/
            app/controllers/
            app/library/
            app/models/
            app/plugins/
            app/views/
        public/
            public/bootstrap/
            public/css/
            public/js/
        schemas/

Как вы уже знаете, Phalcon не навязывает определенную структуру файлов и каталогов для разработки приложений. Этот проект обеспечивает простую стуктуру MVC и корневой каталог public.

После того, как вы откроете приложение в браузере http://localhost/invo вы увидите что-то вроде этого:

.. figure:: ../_static/img/invo-1.png
   :align: center

Приложение состоит из двух частей, фронтенд - внешняя часть, где поситители могут получить информацию о INVO и запросить контактные данные. И бэкенд - административную панель, где зарегистрированный пользователь может управлять своими продуктами и клиентами.

Маршрутизация
-------
INVO использует стандартный маршрутизатор основанный на встроенном компоненте Route. Эти маршруты соответствуют следующим шаблонам: /:controller/:action/:params. Первая часть URI является контроллером, вторая имя действия и остальные параметры.

Маршрут /session/register выполняет контроллер SessionController и его действие registerAction.

Конфигурация
-------------
INVO имеет конфигурационный файл, который устанавливает общие параметры приложения. Этот файл загружается в самом начале
загрузочного файла (public/index.php):

.. code-block:: php

    <?php

    //Read the configuration
    $config = new Phalcon\Config\Adapter\Ini('../app/config/config.ini');

:doc:'Phalcon\\Config <config>' позволяет нам манипулировать файлами в объектно-ориентированного подхода. Файл конфигурации
содержит следующие настройки:

.. code-block:: ini

    [database]
    host     = localhost
    username = root
    password = secret
    name     = invo

    [application]
    controllersDir = /../app/controllers/
    modelsDir      = /../app/models/
    viewsDir       = /../app/views/
    pluginsDir     = /../app/plugins/
    libraryDir     = /../app/library/
    baseUri        = /invo/

    ;[metadata]
    ;adapter = "Apc"
    ;suffix = my-suffix
    ;lifetime = 3600

Phalcon не имеет каких-либо предопределенных соглашений о конфигурациях. Разделы помогут нам организовать необходимые параметры. В этом файле три секции, которые мы будем использовать позже.

Автозагрузчики
-----------
Второе, что видно в в загрузочном файле (public/index.php) это автозагрузчик. Автозагрузчик регистрирует набор
каталогов, где приложение будет искать необходимые классы.

.. code-block:: php

    <?php

    $loader = new \Phalcon\Loader();

    $loader->registerDirs(
        array(
            $config->application->controllersDir,
            $config->application->pluginsDir,
            $config->application->libraryDir,
            $config->application->modelsDir,
        )
    )->register();

Обратите внимание на регистрацию каталогов в файле конфигураций.
Единтсвенная директория которая не была зарегистрирована с помощью автозагрузчика это viewsDir, потому что она не содержит классов, только html + php файлы.

Обработка запроса
--------------------
Пойдем дальше, в конце файла, запрос окончательно обрабатывается с помощью Phalcon\\Mvc\\Application,
этот класс инициализирует и выполняет все что нужно для работы приложения:

.. code-block:: php

    <?php

    $app = new \Phalcon\Mvc\Application($di);

    echo $app->handle()->getContent();

Dependency Injection
--------------------
Посмотрите на первую строку кода на предыдущем блоке, переменная $app получает еще одну переменную $di в своем конструкторе.
Каков смысл этой переменной? Phalcon - слабо связанный фрэймворк, так что нам нужен компонент, который действует как клей, чтобы все работало вместе.
Этот компонент - Phalcon\\DI. Это контейнер, обеспечивающий все связи между частями необходимыми в приложении.

Есть много способов регистрации сервисов в контейнере. В INVO большинство услуг были зарегистрированы с использованием скрытых функций.  Благодаря этому, объекты создаются простейшим образом, уменьшеая ресурсы необходимые для приложения.

Например, в следующем фрагменте, регистрации сессии, анонимная функция будет вызвана только когда приложение требует доступа к данным сессии:

.. code-block:: php

    <?php

    //Начать сессию в первый раз, когда какой нибудь компонент запросит сервис сессий.
    $di->set('session', function() {
        $session = new Phalcon\Session\Adapter\Files();
        $session->start();
        return $session;
    });

Здесь мы можем менять адаптер, выполнить дополнительную инициализацию и многое другое. Обратите внимание, метод был зарегистрирован с помощью имени  "session". Это соглашение позволит фрэймворку идентифицировать активный метод в контейнере.

Запрос имеет множество методов, регистрация каждого метода может быть трудоемкой задачей. По этой причине,
фрэймворк обеспечивает вариант Phalcon\\DI вызывая Phalcon\\DI\\FactoryDefault задачей которого является регистрация
всех методов необходимых фрэймворку.

.. code-block:: php

    <?php

    // FactoryDefault Обеспечивает автоматическую регистрацию
    // полного набора методов необходимых фреймворку
    $di = new \Phalcon\DI\FactoryDefault();

Он регистрирует большинство методов, предусмотренных фрэймворком как стандартные. Если нам надо переопределить
какой либо из методов, мы можем просто определить его снова, как мы делали выше с методом "session". Это причина существования переменной $di.

Log into the Application
------------------------
"Log in" will allow us to work on backend controllers. The separation between backend's controllers and the frontend ones
is only logical. All controllers are located in the same directory (app/controllers/).

To enter into the system, we must have a valid username and password. Users are stored in the table "users"
in the database "invo".

Before we can start a session, we need to configure the connection to the database in the application. A service
called "db" is set up in the service container with that information. As with the autoloader, this time we are
also taking parameters from the configuration file in order to configure a service:

.. code-block:: php

    <?php

    // Database connection is created based on the parameters defined in the configuration file
    $di->set('db', function() use ($config) {
        return new \Phalcon\Db\Adapter\Pdo\Mysql(array(
            "host" => $config->database->host,
            "username" => $config->database->username,
            "password" => $config->database->password,
            "dbname" => $config->database->name
        ));
    });

Here, we return an instance of the MySQL connection adapter. If needed, you could do extra actions such as adding a
logger, a profiler or change the adapter, setting up it as you want.

Back then, the following simple form (app/views/session/index.phtml) requests the logon information. We've removed
some HTML code to make the example more concise:

.. code-block:: html+php

    <?php echo $this->tag->form('session/start') ?>

        <label for="email">Username/Email</label>
        <?php echo $this->tag->textField(array("email", "size" => "30")) ?>

        <label for="password">Password</label>
        <?php echo $this->tag->passwordField(array("password", "size" => "30")) ?>

        <?php echo $this->tag->submitButton(array('Login')) ?>

    </form>

The SessionController::startAction (app/controllers/SessionController.phtml) has the task of validate the
data entered checking for a valid user in the database:

.. code-block:: php

    <?php

    class SessionController extends ControllerBase
    {

        // ...

        private function _registerSession($user)
        {
            $this->session->set('auth', array(
                'id' => $user->id,
                'name' => $user->name
            ));
        }

        public function startAction()
        {
            if ($this->request->isPost()) {

                //Receiving the variables sent by POST
                $email = $this->request->getPost('email', 'email');
                $password = $this->request->getPost('password');

                $password = sha1($password);

                //Find for the user in the database
                $user = Users::findFirst(array(
                    "email = :email: AND password = :password: AND active = 'Y'",
                    "bind" => array('email' => $email, 'password' => $password)
                ));
                if ($user != false) {

                    $this->_registerSession($user);

                    $this->flash->success('Welcome ' . $user->name);

                    //Forward to the 'invoices' controller if the user is valid
                    return $this->dispatcher->forward(array(
                        'controller' => 'invoices',
                        'action' => 'index'
                    ));
                }

                $this->flash->error('Wrong email/password');
            }

            //Forward to the login form again
            return $this->dispatcher->forward(array(
                'controller' => 'session',
                'action' => 'index'
            ));

        }

    }

For simplicity, we have used "sha1_" to store the password hashes in the database, however, this algorithm is
not recommended in real applications, use " :doc:`bcrypt <security>`" instead.

Note that multiple public attributes are accessed in the controller like: $this->flash, $this->request or $this->session.
These are services defined in services container from earlier. When they're accessed the first time, are injected as part
of the controller.

These services are shared, which means that we are always accessing the same instance regardless of the place
where we invoke them.

For instance, here we invoke the "session" service and then we store the user identity in the variable "auth":

.. code-block:: php

    <?php

    $this->session->set('auth', array(
        'id' => $user->id,
        'name' => $user->name
    ));

Securing the Backend
--------------------
The backend is a private area where only registered users have access. Therefore, it is necessary to check that only
registered users have access to these controllers. If you aren't logged in the application and you try to access,
for example, the products controller (that is private) you will see a screen like this:

.. figure:: ../_static/img/invo-2.png
   :align: center

Every time someone attempts to access any controller/action, the application verifies that the current role (in session)
has access to it, otherwise it displays a message like the above and forwards the flow to the home page.

Now let's find out how the application accomplishes this. The first thing to know is that there is a component called
:doc:`Dispatcher <dispatching>`. It is informed about the route found by the :doc:`Routing <routing>` component. Then,
it is responsible for loading the appropriate controller and execute the corresponding action method.

Normally, the framework creates the Dispatcher automatically. In our case, we want to perform a verification
before executing the required action, checking if the user has access to it or not. To achieve this, we have
replaced the component by creating a function in the bootstrap:

.. code-block:: php

    <?php

    $di->set('dispatcher', function() use ($di) {
        $dispatcher = new Phalcon\Mvc\Dispatcher();
        return $dispatcher;
    });

We now have total control over the Dispatcher used in the application. Many components in the framework trigger
events that allow us to modify their internal flow of operation. As the dependency Injector component acts as glue
for components, a new component called :doc:`EventsManager <events>` aids us to intercept the events produced
by a component routing the events to listeners.

Events Management
^^^^^^^^^^^^^^^^^
A :doc:`EventsManager <events>` allows us to attach listeners to a particular type of event. The type that
interest us now is "dispatch", the following code filters all events produced by the Dispatcher:

.. code-block:: php

    <?php

    $di->set('dispatcher', function() use ($di) {

        //Obtain the standard eventsManager from the DI
        $eventsManager = $di->getShared('eventsManager');

        //Instantiate the Security plugin
        $security = new Security($di);

        //Listen for events produced in the dispatcher using the Security plugin
        $eventsManager->attach('dispatch', $security);

        $dispatcher = new Phalcon\Mvc\Dispatcher();

        //Bind the EventsManager to the Dispatcher
        $dispatcher->setEventsManager($eventsManager);

        return $dispatcher;
    });

The Security plugin is a class located at (app/plugins/Security.php). This class implements the method
"beforeExecuteRoute". This is the same name as one of the events produced in the Dispatcher:

.. code-block:: php

    <?php

    use Phalcon\Events\Event,
        Phalcon\Mvc\Dispatcher,
        Phalcon\Mvc\User\Plugin;

    class Security extends Plugin
    {

        // ...

        public function beforeExecuteRoute(Event $event, Dispatcher $dispatcher)
        {
            // ...
        }

    }

The hooks events always receive a first parameter that contains contextual information of the event produced ($event)
and a second one that is the object that produced the event itself ($dispatcher). It is not mandatory that
plugins extend the class Phalcon\\Mvc\\User\\Plugin, but by doing this they gain easier access to the services
available in the application.

Now, we're verifying the role in the current session, checking if he/she has access using the ACL list.
If he/she does not have access we redirect him/her to the home screen as explained before:

.. code-block:: php

    <?php

    use Phalcon\Events\Event,
        Phalcon\Mvc\Dispatcher,
        Phalcon\Mvc\User\Plugin;

    class Security extends Plugin
    {

        // ...

        public function beforeExecuteRoute(Event $event, Dispatcher $dispatcher)
        {

            //Check whether the "auth" variable exists in session to define the active role
            $auth = $this->session->get('auth');
            if (!$auth) {
                $role = 'Guests';
            } else {
                $role = 'Users';
            }

            //Take the active controller/action from the dispatcher
            $controller = $dispatcher->getControllerName();
            $action = $dispatcher->getActionName();

            //Obtain the ACL list
            $acl = $this->_getAcl();

            //Check if the Role have access to the controller (resource)
            $allowed = $acl->isAllowed($role, $controller, $action);
            if ($allowed != Phalcon\Acl::ALLOW) {

                //If he doesn't have access forward him to the index controller
                $this->flash->error("You don't have access to this module");
                $dispatcher->forward(
                    array(
                        'controller' => 'index',
                        'action' => 'index'
                    )
                );

                //Returning "false" we tell to the dispatcher to stop the current operation
                return false;
            }

        }

    }

Providing an ACL list
^^^^^^^^^^^^^^^^^^^^^
In the above example we have obtained the ACL using the method $this->_getAcl(). This method is also
implemented in the Plugin. Now we are going to explain step-by-step how we built the access control list (ACL):

.. code-block:: php

    <?php

    //Create the ACL
    $acl = new Phalcon\Acl\Adapter\Memory();

    //The default action is DENY access
    $acl->setDefaultAction(Phalcon\Acl::DENY);

    //Register two roles, Users is registered users
    //and guests are users without a defined identity
    $roles = array(
        'users' => new Phalcon\Acl\Role('Users'),
        'guests' => new Phalcon\Acl\Role('Guests')
    );
    foreach ($roles as $role) {
        $acl->addRole($role);
    }

Now we define the resources for each area respectively. Controller names are resources and their actions are
accesses for the resources:

.. code-block:: php

    <?php

    //Private area resources (backend)
    $privateResources = array(
      'companies' => array('index', 'search', 'new', 'edit', 'save', 'create', 'delete'),
      'products' => array('index', 'search', 'new', 'edit', 'save', 'create', 'delete'),
      'producttypes' => array('index', 'search', 'new', 'edit', 'save', 'create', 'delete'),
      'invoices' => array('index', 'profile')
    );
    foreach ($privateResources as $resource => $actions) {
        $acl->addResource(new Phalcon\Acl\Resource($resource), $actions);
    }

    //Public area resources (frontend)
    $publicResources = array(
      'index' => array('index'),
      'about' => array('index'),
      'session' => array('index', 'register', 'start', 'end'),
      'contact' => array('index', 'send')
    );
    foreach ($publicResources as $resource => $actions) {
        $acl->addResource(new Phalcon\Acl\Resource($resource), $actions);
    }

The ACL now have knowledge of the existing controllers and their related actions. Role "Users" has access to
all the resources of both frontend and backend. The role "Guests" only has access to the public area:

.. code-block:: php

    <?php

    //Grant access to public areas to both users and guests
    foreach ($roles as $role) {
        foreach ($publicResources as $resource => $actions) {
            $acl->allow($role->getName(), $resource, '*');
        }
    }

    //Grant access to private area only to role Users
    foreach ($privateResources as $resource => $actions) {
        foreach ($actions as $action) {
            $acl->allow('Users', $resource, $action);
        }
    }

Hooray!, the ACL is now complete.

User Components
---------------
All the UI elements and visual style of the application has been achieved mostly through `Twitter Bootstrap`_.
Some elements, such as the navigation bar changes according to the state of the application. For example, in the
upper right corner, the link "Log in / Sign Up" changes to "Log out" if an user is logged into the application.

This part of the application is implemented in the component "Elements" (app/library/Elements.php).

.. code-block:: php

    <?php

    use Phalcon\Mvc\User\Component;

    class Elements extends Component
    {

        public function getMenu()
        {
            //...
        }

        public function getTabs()
        {
            //...
        }

    }

This class extends the Phalcon\\Mvc\\User\\Component, it is not imposed to extend a component with this class, but
it helps to get access more quickly to the application services. Now, we register this class in the services container:

.. code-block:: php

    <?php

    //Register an user component
    $di->set('elements', function(){
        return new Elements();
    });

As controllers, plugins or components within a view, this component also has access to the services registered
in the container and by just accessing an attribute with the same name as a previously registered service:

.. code-block:: html+php

    <div class="navbar navbar-fixed-top">
        <div class="navbar-inner">
            <div class="container">
                <a class="btn btn-navbar" data-toggle="collapse" data-target=".nav-collapse">
                    <span class="icon-bar"></span>
                    <span class="icon-bar"></span>
                    <span class="icon-bar"></span>
                </a>
                <a class="brand" href="#">INVO</a>
                <?php echo $this->elements->getMenu() ?>
            </div>
        </div>
    </div>

    <div class="container">
        <?php echo $this->getContent() ?>
        <hr>
        <footer>
            <p>&copy; Company 2012</p>
        </footer>
    </div>

The important part is:

.. code-block:: html+php

    <?php echo $this->elements->getMenu() ?>

Working with the CRUD
---------------------
Most options that manipulate data (companies, products and types of products), were developed using a basic and
common CRUD_ (Create, Read, Update and Delete). Each CRUD contains the following files:

.. code-block:: bash

    invo/
        app/
            app/controllers/
                ProductsController.php
            app/models/
                Products.php
            app/views/
                products/
                    edit.phtml
                    index.phtml
                    new.phtml
                    search.phtml

Each controller has the following actions:

.. code-block:: php

    <?php

    class ProductsController extends ControllerBase
    {

        /**
         * The start action, it shows the "search" view
         */
        public function indexAction()
        {
            //...
        }

        /**
         * Execute the "search" based on the criteria sent from the "index"
         * Returning a paginator for the results
         */
        public function searchAction()
        {
            //...
        }

        /**
         * Shows the view to create a "new" product
         */
        public function newAction()
        {
            //...
        }

        /**
         * Shows the view to "edit" an existing product
         */
        public function editAction()
        {
            //...
        }

        /**
         * Creates a product based on the data entered in the "new" action
         */
        public function createAction()
        {
            //...
        }

        /**
         * Updates a product based on the data entered in the "edit" action
         */
        public function saveAction()
        {
            //...
        }

        /**
         * Deletes an existing product
         */
        public function deleteAction($id)
        {
            //...
        }

    }

The Search Form
^^^^^^^^^^^^^^^
Every CRUD starts with a search form. This form shows each field that has the table (products), allowing the user
creating a search criteria from any field. Table "products" has a relationship to the table "products_types".
In this case, we previously queried the records in this table in order to facilitate the search by that field:

.. code-block:: php

    <?php

    /**
     * The start action, it shows the "search" view
     */
    public function indexAction()
    {
        $this->persistent->searchParams = null;
        $this->view->productTypes = ProductTypes::find();
    }

All the "product types" are queried and passed to the view as a local variable "productTypes". Then, in the view
(app/views/index.phtml) we show a "select" tag filled with those results:

.. code-block:: html+php

    <div>
        <label for="product_types_id">Product Type</label>
        <?php echo $this->tag->select(array(
            "product_types_id",
            $productTypes,
            "using" => array("id", "name"),
            "useDummy" => true
        )) ?>
    </div>

Note that $productTypes contains the data necessary to fill the SELECT tag using Phalcon\\Tag::select. Once the form
is submitted, the action "search" is executed in the controller performing the search based on the data entered by
the user.

Performing a Search
^^^^^^^^^^^^^^^^^^^
The action "search" has a dual behavior. When accessed via POST, it performs a search based on the data sent from the
form. But when accessed via GET it moves the current page in the paginator. To differentiate one from another HTTP method,
we check it using the :doc:`Request <request>` component:

.. code-block:: php

    <?php

    /**
     * Execute the "search" based on the criteria sent from the "index"
     * Returning a paginator for the results
     */
    public function searchAction()
    {

        if ($this->request->isPost()) {
            //create the query conditions
        } else {
            //paginate using the existing conditions
        }

        //...

    }

With the help of :doc:`Phalcon\\Mvc\\Model\\Criteria <../api/Phalcon_Mvc_Model_Criteria>`, we can create the search
conditions intelligently based on the data types and values sent from the form:

.. code-block:: php

    <?php

    $query = Criteria::fromInput($this->di, "Products", $_POST);

This method verifies which values are different from "" (empty string) and null and takes them into account to create
the search criteria:

* If the field data type is text or similar (char, varchar, text, etc.) It uses an SQL "like" operator to filter the results.
* If the data type is not text or similar, it'll use the operator "=".

Additionally, "Criteria" ignores all the $_POST variables that do not match any field in the table.
Values are automatically escaped using "bound parameters".

Now, we store the produced parameters in the controller's session bag:

.. code-block:: php

    <?php

    $this->persistent->searchParams = $query->getParams();

A session bag, is a special attribute in a controller that persists between requests. When accessed, this attribute injects
a :doc:`Phalcon\\Session\\Bag <../api/Phalcon_Session_Bag>` service that is independent in each controller.

Then, based on the built params we perform the query:

.. code-block:: php

    <?php

    $products = Products::find($parameters);
    if (count($products) == 0) {
        $this->flash->notice("The search did not found any products");
        return $this->forward("products/index");
    }

If the search doesn't return any product, we forward the user to the index action again. Let's pretend the
search returned results, then we create a paginator to navigate easily through them:

.. code-block:: php

    <?php

    $paginator = new Phalcon\Paginator\Adapter\Model(array(
        "data" => $products,    //Data to paginate
        "limit" => 5,           //Rows per page
        "page" => $numberPage   //Active page
    ));

    //Get active page in the paginator
    $page = $paginator->getPaginate();

Finally we pass the returned page to view:

.. code-block:: php

    <?php

    $this->view->setVar("page", $page);

In the view (app/views/products/search.phtml), we traverse the results corresponding to the current page:

.. code-block:: html+php

    <?php foreach ($page->items as $product) { ?>
        <tr>
            <td><?= $product->id ?></td>
            <td><?= $product->getProductTypes()->name ?></td>
            <td><?= $product->name ?></td>
            <td><?= $product->price ?></td>
            <td><?= $product->active ?></td>
            <td><?= $this->tag->linkTo("products/edit/" . $product->id, 'Edit') ?></td>
            <td><?= $this->tag->linkTo("products/delete/" . $product->id, 'Delete') ?></td>
        </tr>
    <?php } ?>

Creating and Updating Records
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Now let's see how the CRUD creates and updates records. From the "new" and "edit" views the data entered by the user
are sent to the actions "create" and "save" that perform actions of "creating" and "updating" products respectively.

In the creation case, we recover the data submitted and assign them to a new "products" instance:

.. code-block:: php

    <?php

    /**
     * Creates a product based on the data entered in the "new" action
     */
    public function createAction()
    {

        $products = new Products();

        $products->id = $this->request->getPost("id", "int");
        $products->product_types_id = $this->request->getPost("product_types_id", "int");
        $products->name = $this->request->getPost("name", "striptags");
        $products->price = $this->request->getPost("price", "double");
        $products->active = $this->request->getPost("active");

        //...

    }

Data is filtered before being assigned to the object. This filtering is optional, the ORM escapes the input data and
performs additional casting according to the column types.

When saving we'll know whether the data conforms to the business rules and validations implemented in the model Products:

.. code-block:: php

    <?php

    /**
     * Creates a product based on the data entered in the "new" action
     */
    public function createAction()
    {

        //...

        if (!$products->create()) {

            //The store failed, the following messages were produced
            foreach ($products->getMessages() as $message) {
                $this->flash->error((string) $message);
            }
            return $this->forward("products/new");

        } else {
            $this->flash->success("Product was created successfully");
            return $this->forward("products/index");
        }

    }

Now, in the case of product updating, first we must present to the user the data that is currently in the edited record:

.. code-block:: php

    <?php

    /**
     * Shows the view to "edit" an existing product
     */
    public function editAction($id)
    {

        //...

        $product = Products::findFirstById($id);

        $this->tag->setDefault("id", $product->id);
        $this->tag->setDefault("product_types_id", $product->product_types_id);
        $this->tag->setDefault("name", $product->name);
        $this->tag->setDefault("price", $product->price);
        $this->tag->setDefault("active", $product->active);

    }

The "setDefault" helper sets a default value in the form on the attribute with the same name. Thanks to this,
the user can change any value and then sent it back to the database through to the "save" action:

.. code-block:: php

    <?php

    /**
     * Updates a product based on the data entered in the "edit" action
     */
    public function saveAction()
    {

        //...

        //Find the product to update
        $id = $this->request->getPost("id");
        $product = Products::findFirstById($id);
        if (!$product) {
            $this->flash->error("products does not exist " . $id);
            return $this->forward("products/index");
        }

        //... assign the values to the object and store it

    }

Changing the Title Dynamically
------------------------------
When you browse between one option and another will see that the title changes dynamically indicating where
we are currently working. This is achieved in each controller initializer:

.. code-block:: php

    <?php

    class ProductsController extends ControllerBase
    {

        public function initialize()
        {
            //Set the document title
            $this->tag->setTitle('Manage your product types');
            parent::initialize();
        }

        //...

    }

Note, that the method parent::initialize() is also called, it adds more data to the title:

.. code-block:: php

    <?php

    class ControllerBase extends Phalcon\Mvc\Controller
    {

        protected function initialize()
        {
            //Prepend the application name to the title
            $this->tag->prependTitle('INVO | ');
        }

        //...
    }

Finally, the title is printed in the main view (app/views/index.phtml):

.. code-block:: html+php

    <!DOCTYPE html>
    <html>
        <head>
            <?php echo $this->tag->getTitle() ?>
        </head>
        <!-- ... -->
    </html>

Conclusion
----------
This tutorial covers many more aspects of building applications with Phalcon, hope you have served to
learn more and get more out of the framework.

.. _Github: https://github.com/phalcon/invo
.. _CRUD: http://en.wikipedia.org/wiki/Create,_read,_update_and_delete
.. _Twitter Bootstrap: http://twitter.github.io/bootstrap/
.. _sha1: http://php.net/manual/en/function.sha1.php
.. _bcrypt: http://stackoverflow.com/questions/4795385/how-do-you-use-bcrypt-for-hashing-passwords-in-php
