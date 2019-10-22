## Laravel Introduction
### 1. Object-relationship Mapping (eloquent)
![Eloquent & Query Builder](https://github.com/seekerliu/laravel-tips/raw/master/figures/introduction-to-query-builder-and-eloquent-1.png)
#### Model 
 - is an object to present actual schema of database table; It use object property maps the field of database table.
 - can build relationship with other tables by using default method. Just like foreign key.
#### Query Builder
 - is special calss provides an query interface for ORM Model.
 - can cover both eloquent query method and original SQL statment.
#### Coding Usage
 - Accessors & Mutators
     ```
     class User extends Model
    {
        public function getFirstNameAttribute($value)
        {
            return ucfirst($value);
        }
        public function getFullNameAttribute()
        {
            return "{$this->first_name} {$this->last_name}";
        }
    }
    
    $user = App\User::find(1);
    $firstName = $user->first_name;
    $user->first_name = 'Sally';
     ```
### 2. DI & IOC
 + factory module
 ```
    class SuperModuleFactory
    {
        public function makeModule($moduleName, $options)
        {
            switch ($moduleName) {
                case 'Fight': 
                    return new Fight($options[0], $options[1]);
                case 'Force': 
                    return new Force($options[0]);
                case 'Shot': 
                    return new Shot($options[0], $options[1], $options[2]);
            }
        }
    }
    class Autobot
    {
        protected $power;

        public function __construct(array $modules)
        {
            // init factory
            $factory = new SuperModuleFactory;

            // make modules
            foreach ($modules as $moduleName => $moduleOptions) {
                $this->power[] = $factory->makeModule($moduleName, $moduleOptions);
            }
        }
    }
    $bee = new Autobot([
        'Fight' => [9, 100],
        'Shot' => [99, 50, 2]
    ]);
 ```
 + Dependency Inject
 ```
    // Build an interface
    interface SuperModuleInterface
    {
        public function activate(array $target);
    }
    
    // BUild ability which implement SuperModuleInterface
    class Xray implements SuperModuleInterface
    {
        public function activate(array $target)
        {
            // do something
        }
    }
    
    class Bomb implements SuperModuleInterface
    {
        public function activate(array $target)
        {
            // do something
        }
    }
    
    // Autobot use different abilities which has unified interface to plugin
    class Autobot
    {
        protected $module;

        public function __construct(SuperModuleInterface $module)
        {
            $this->module = $module;
        }
        public function execute(Enamy $target)
        {
            $this->module->activate($target);
        }
    } 
    
    // Super Ability
    $superModule = new Xray;
    // Plugin to an autobot
    $Autobot = new Autobot($superModule);
 ```
 + Service Container 
 
   Dependency injection is a fancy phrase that essentially means this: class dependencies are "injected" into the class via the constructor or, in some cases, "setter" methods.
```
    class Mailer
    {
        public function mail($recipient, $content)
        {
            // Send an email to the recipient
        }
    }
    
    class UserManager
    {
        private $mailer;

        public function __construct(Mailer $mailer)  // here inject an object with type hint
        {
            $this->mailer = $mailer;
        }

        public function register($email, $password)
        {
            // create user & send an email
            $this->mailer->mail($email, 'Hello and welcome!');
        }
    }
    
    use Illuminate\Container\Container;

    $container = Container::getInstance();
   
    // $userManager = new UserManager(new Mailer);
    $userManager = $container->make(UserManager::class);
    
    $userManager->register('dave@davejamesmiller.com', 'MySuperSecurePassword!');
```
 
 + Inversion of Control
         
   Now we inverted the control. Instead of relying on a concrete instance, we can now inject any instance that consumes the type hinted interface. The interface takes care that the later concrete instance implements all the methods that we are going to use, so that we can still rely on them in the dependent classes.
 ```
    class Container
    {
        protected $binds;

        protected $instances;

        public function bind($abstract, $concrete)
        {
            if ($concrete instanceof Closure) {
                $this->binds[$abstract] = $concrete;
            } else {
                $this->instances[$abstract] = $concrete;
            }
        }

        public function make($abstract, $parameters = [])
        {
            if (isset($this->instances[$abstract])) {
                return $this->instances[$abstract];
            }

            array_unshift($parameters, $this);

            return call_user_func_array($this->binds[$abstract], $parameters);
        }
    }
    
    // make instance of Container
    $container = new Container;

    // Bind Closure to Autobot classname
    $container->bind('autobot', function($container, $moduleName) {
        return new Autobot($container->make($moduleName));
    });

    // Bind Closure to ability factory class
    $container->bind('xray', function($container) {
        return new Xray;
    });
    $container->bind('bomb', function($container) {
        return new Bomb;
    });
    
    // try
    $autobot_1 = $container->make('autobot', 'xray');
    $autobot_2 = $container->make('autobot', 'bomb');
    $autobot_3 = $container->make('autobot', 'xray');
 ```
 + ServiceProvider
 
    All Laravel's services are bootstrapped via service providers.
 ```
    class TransformService extends ServiceProvider
    {
        public function register()
        {
            $container->bind('autobot', function($container, $moduleName) {
                return new Superman($container->make($moduleName));
            });

            // Bind Closure to ability factory class
            $container->bind('xray', function($container) {
                return new Xray;
            });
            $container->bind('bomb', function($container) {
                return new Bomb;
            });
        }
        public function boot()
        {
            $autobot_1 = $container->make('autobot', 'xray');
            // $autobot_2 = $container->make('autobot', 'bomb');
            // $autobot_3 = $container->make('autobot', 'xray');
            $autobot_1->activate($target);
        }
    }
 ```
 + bind() method
 
    maps the abstract name with specified Closure or object
 ```
    // define two interfaces
    interface MyInterface { /* ... */ }
    interface AnotherInterface { /* ... */ }

    // Myclass implement MyInterface and its dependency implements another interface
    class MyClass implements MyInterface
    {
        private $dependency;

        // 依赖了一个实现 AnotherInterface 接口的类的实例
        public function __construct(AnotherInterface $dependency)
        {
            $this->dependency = $dependency;
        }
    }

    // map each interface with specified class
    $container->bind(MyInterface::class, MyClass::class);
    $container->bind(AnotherInterface::class, AnotherClass::class);

    $instance = $container->make(MyInterface::class);
 ```
 + multiple bind
 ```
    use Illuminate\Support\Facades\Storage;
    use App\Http\Controllers\PhotoController;
    use App\Http\Controllers\VideoController;
    use Illuminate\Contracts\Filesystem\Filesystem;

    $this->app->when(PhotoController::class)
             ->needs(Filesystem::class)
             ->give(function () {
                 return Storage::disk('local');
             });

    $this->app->when(VideoController::class)
             ->needs(Filesystem::class)
             ->give(function () {
                 return Storage::disk('s3');
             });
 ```
### 3. Closure
 - bit similar with anonymous function in Javascript
 - has its own class Closure
 ```
   Closure {   
       public static Closure bind (Closure $closure , object $newthis [, mixed $newscope = 'static' ])  
       public Closure bindTo (object $newthis [, mixed $newscope = 'static' ])  
   }
 ```
 - bind()
 ```
    <?php
    class A {
        private static $sfoo = 1;
        private $ifoo = 2;
    }
    $cl1 = static function() {
        return A::$sfoo;
    };
    $cl2 = function() {
        return $this->ifoo;
    };

    $bcl1 = Closure::bind($cl1, null, 'A');
    $bcl2 = Closure::bind($cl2, new A(), 'A');
    echo $bcl1(), "\n";
    echo $bcl2(), "\n";
    ?>
  ```
  - bindTo()
  ```
    <?php
    class A {
        function __construct($val) {
            $this->val = $val;
        }
        function getClosure() {
            //returns closure bound to this object and scope
            return function() { return $this->val; };
        }
    }

    $ob1 = new A(1);
    $ob2 = new A(2);

    $cl = $ob1->getClosure();
    echo $cl(), "\n";
    $cl = $cl->bindTo($ob2);
    echo $cl(), "\n";
    ?>
  ```
  - bindTo used on class property setting
  ```
    <?php
    trait DynamicDefinition {

        public function __call($name, $args) {
            if (is_callable($this->$name)) {
                return call_user_func($this->$name, $args);
            }
            else {
                throw new \RuntimeException("Method {$name} does not exist");
            }
        }

        public function __set($name, $value) {
            $this->$name = is_callable($value)?
                $value->bindTo($this, $this):
                $value;
        }
    }

    class Foo {
        use DynamicDefinition;
        private $privateValue = 'I am private';
    }

    $foo = new Foo;
    $foo->bar = function() {
        return $this->privateValue;
    };

    // prints 'I am private'
    print $foo->bar();

    ?>
  ```
