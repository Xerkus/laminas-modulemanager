# The Module Manager

The module manager, `Laminas\ModuleManager\ModuleManager`, is responsible for
iterating over an array of module names and triggering a sequence of events for
each.  Instantiation of module classes, initialization tasks, and configuration
are all performed by attached event listeners.

## Module Manager Events

The Module Manager events are defined in `Laminas\ModuleManager\ModuleEvent`.

### loadModules (`ModuleEvent::EVENT_LOAD_MODULES`)

This event is primarily used internally to help encapsulate the work of loading
modules in event listeners, and allows the `loadModules.post` event to be more
user-friendly. Internal listeners will attach to this event with a negative
priority instead of `loadModules.post` so that users can safely assume things like
config merging have been done once `loadModules.post` is triggered, without having
to worry about priorities.

### loadModule.resolve (`ModuleEvent::EVENT_LOAD_MODULE_RESOLVE`)

Triggered for each module that is to be loaded. The listener(s) to this event
are responsible for taking a module name and resolving it to an instance of some
class. The default module resolver looks for the class `{modulename}\Module`,
instantiating and returning it if it exists.

The name of the module may be retrieved by listeners using the `getModuleName()`
method of the `Event` object; a listener should then take that name and resolve
it to an object instance representing the given module. Multiple listeners can
be attached to this event, and the module manager will trigger them in order of
their priority until one returns an object. This allows you to attach additional
listeners which have alternative methods of resolving modules from a given
module name.

### loadModule (`ModuleEvent::EVENT_LOAD_MODULE`)

Once a module resolver listener has resolved the module name to an object, the
module manager then triggers this event, passing the newly created object to all
listeners.

### mergeConfig (`ModuleEvent::EVENT_MERGE_CONFIG`)

After all modules have been loaded, the `mergeConfig` event is triggered. By default,
`Laminas\ModuleManager\Listener\ConfigLister` listens on this event at priority 1000, and merges all
configuration. You may attach additional listeners to this event in order to manipulate the merged
configuration. See [the tutorial on manipulating merged configuration](https://getlaminas.org/manual/current/en/tutorials/config.advanced.html#manipulating-merged-configuration)
for more information.

### loadModules.post (`ModuleEvent::EVENT_LOAD_MODULES_POST`)

This event is triggered by the module manager to allow any listeners to perform
work after every module has finished loading. For example, the default
configuration listener, `Laminas\ModuleManager\Listener\ConfigListener` (covered
later), attaches to this event to merge additional user-supplied configuration
which is meant to override the default supplied configurations of installed
modules.

## Module Manager Listeners

By default, Laminas provides several useful module manager listeners. All
shipped listeners are in the `Laminas\ModuleManager\Listener` namespace.

### DefaultListenerAggregate

To address the most common use case of the module manager, Laminas provides
this default aggregate listener. In most cases, this will be the only listener
you will need to attach to use the module manager, as it will take care of
properly attaching the requisite listeners (those listed below) for the module
system to function properly.

### AutoloaderListener

This listener checks each module to see if it has implemented
`Laminas\ModuleManager\Feature\AutoloaderProviderInterface` or defined the
`getAutoloaderConfig()` method. If so, it calls the `getAutoloaderConfig()`
method on the module class and passes the returned array to
`Laminas\Loader\AutoloaderFactory`.

### ModuleDependencyCheckerListener

This listener checks each module to verify if all the modules it depends on were
loaded. When a module class implements
`Laminas\ModuleManager\Feature\DependencyIndicatorInterface` or has defined the
`getModuleDependencies()` method, the listener will call
`getModuleDependencies()`. Each of the values returned by the method is checked
against the loaded modules list: if one of the values is not in that list, a
`Laminas\ModuleManager\Exception\MissingDependencyModuleException` is thrown.

### ConfigListener

If a module class has a `getConfig()` method, or implements
`Laminas\ModuleManager\Feature\ConfigProviderInterface`, this listener will call it
and merge the returned array (or `Traversable` object) into the main application
configuration.

### InitTrigger

If a module class either implements `Laminas\ModuleManager\Feature\InitProviderInterface`,
or defines an `init()` method, this listener will call `init()` and pass the
current instance of `Laminas\ModuleManager\ModuleManager` as the sole parameter.

Like the `OnBootstrapListener`, the `init()` method is called for **every**
module implementing this feature, on **every** page request and should **only**
be used for performing **lightweight** tasks such as registering event
listeners.

### LocatorRegistrationListener

If a module class implements `Laminas\ModuleManager\Feature\LocatorRegisteredInterface`,
this listener will inject the module class instance into the `ServiceManager`
using the module class name as the service name. This allows you to later
retrieve the module class from the `ServiceManager`.

### ModuleResolverListener

This is the default module resolver. It attaches to the `loadModule.resolve`
event and returns an instance of `{moduleName}\Module`.

Since 2.8.0, if the module name provided resolves to a fully qualified class
name, it returns that verbatim.

### OnBootstrapListener

If a module class implements
`Laminas\ModuleManager\Feature\BootstrapListenerInterface`, or defines an
`onBootstrap()` method, this listener will register the `onBootstrap()` method
with the `Laminas\Mvc\Application` `bootstrap` event. This method will then be
triggered during the `bootstrap` event (and passed an `MvcEvent` instance).

Like the `InitTrigger`, the `onBootstrap()` method is called for **every**
module implementing this feature, on **every** page request, and should **only**
be used for performing **lightweight** tasks such as registering event
listeners.

### ServiceListener

If a module class implements `Laminas\ModuleManager\Feature\ServiceProviderInterface`,
or defines an `getServiceConfig()` method, this listener will call that method
and aggregate the return values for use in configuring the `ServiceManager`.

The `getServiceConfig()` method may return either an array of configuration
compatible with `Laminas\ServiceManager\Config`, an instance of that class, or the
string name of a class that extends it. Values are merged and aggregated on
completion, and then merged with any configuration from the `ConfigListener`
falling under the `service_manager` key. For more information, see the
`ServiceManager` documentation.

Unlike the other listeners, this listener is not managed by the `DefaultListenerAggregate`;
instead, it is created and instantiated within the
`Laminas\Mvc\Service\ModuleManagerFactory`, where it is injected with the current
`ServiceManager` instance before being registered with the `ModuleManager`
events.

Additionally, this listener manages a variety of plugin managers, including
[view helpers](https://github.com/laminas/laminas-view),
[controllers](https://docs.laminas.dev/laminas-mvc/controllers/), and
[controller plugins](https://docs.laminas.dev/laminas-mvc/plugins/). In each case,
you may either specify configuration to define plugins, or provide configuration via a `Module`
class. Configuration follows the same format as for the `ServiceManager`. The following table
outlines the plugin managers that may be configured this way (including the `ServiceManager`), the
configuration key to use, the `ModuleManager` feature interface to optionally implement (all
interfaces specified live in the `Laminas\ModuleManager\Feature` namespace) , and the module method to
optionally define to provide configuration.

Plugin Manager | Config Key | Interface | Module Method
---------------|------------|-----------|--------------
`Laminas\Mvc\Controller\ControllerManager` | `controllers` | `ControllerProviderInterface` | `getControllerConfig`
`Laminas\Mvc\Controller\PluginManager` | `controller_plugins` | `ControllerPluginProviderInterface` | `getControllerPluginConfig`
`Laminas\Filter\FilterPluginManager` | `filters` | `FilterProviderInterface` | `getFilterConfig`
`Laminas\Form\FormElementManager` | `form_elements` | `FormElementProviderInterface` | `getFormElementConfig`
`Laminas\Stdlib\Hydrator\HydratorPluginManager` | `hydrators` | `HydratorProviderInterface` | `getHydratorConfig`
`Laminas\InputFilter\InputFilterPluginManager` | `input_filters` | `InputFilterProviderInterface` | `getInputFilterConfig`
`Laminas\Mvc\Router\RoutePluginManager` | `route_manager` | `RouteProviderInterface` | `getRouteConfig`
`Laminas\Serializer\AdapterPluginManager` | `serializers` | `SerializerProviderInterface` | `getSerializerConfig`
`Laminas\ServiceManager\ServiceManager` | `service_manager` | `ServiceProviderInterface` | `getServiceConfig`
`Laminas\Validator\ValidatorPluginManager` | `validators` | `ValidatorProviderInterface` | `getValidatorConfig`
`Laminas\View\HelperPluginManager` | `view_helpers` | `ViewHelperProviderInterface` | `getViewHelperConfig`
`Laminas\Log\ProcessorPluginManager` | `log_processors` | `LogProcessorProviderInterface` | `getLogProcessorConfig`
`Laminas\Log\WriterPluginManager` | `log_writers` | `LogWriterProviderInterface` | `getLogWriterConfig`

Configuration follows the examples in the ServiceManager
[configuration section](https://docs.laminas.dev/laminas-servicemanager/configuring-the-service-manager/).
As a brief recap, the following configuration keys and values are allowed:

Config Key | Allowed values
-----------|---------------
`services` | service name/instance pairs (these should likely be defined only in `Module` classes
`invokables` | service name/class name pairs of classes that may be invoked without constructor arguments (deprecated with laminas-servicemanager v3; use the `InvokableFactory` instead)
`factories` | service names pointing to factories. Factories may be any PHP callable, or a string class name of a class implementing `Laminas\ServiceManager\FactoryInterface`, or of a class implementing the `__invoke` method (if a callable is used, it should be defined only in `Module` classes)
`abstract_factories` | array of either concrete instances of `Laminas\ServiceManager\AbstractFactoryInterface`, or string class names of classes implementing that interface (if an instance is used, it should be defined only in `Module` classes)
`initializers` | array of PHP callables or string class names of classes implementing `Laminas\ServiceManager\InitializerInterface` (if a callable is used, it should be defined only in `Module` classes)

When working with plugin managers, you will be passed the plugin manager instance to factories,
abstract factories, and initializers. If you need access to the application services, you can use
the `getServiceLocator()` method, as in the following example:

```php
public function getViewHelperConfig()
{
    return ['factories' => [
        'foo' => function ($helpers) {
            $container   = $helpers->getServiceLocator();
            $someService = $container->get('SomeService');
            $helper      = new Helper\Foo($someService);

            return $helper;
        },
    ]];
}
```

This is a powerful technique, as it allows your various plugins to remain
agnostic with regards to where and how dependencies are injected, and thus
allows you to use Inversion of Control principals even with plugins.

> #### Factories with laminas-servicemanager v3
>
> Starting in the v3 releases of laminas-servicemanager, factories invoked by
> plugin managers now receive the *parent* container, and not the plugin manager
> itself. You can write your factories under v2, and prepare them for v3, by
> testing the incoming argument to see if it is a plugin manager:
>
> ```php
> use Laminas\ServiceManager\AbstractPluginManager;
>
> function ($container) {
>     $container = $container instanceof AbstractPluginManager
>         ? $container->getServiceLocator()
>         : $container;
>
>     // create instance with dependencies pulled from app container...
> }
> ```
