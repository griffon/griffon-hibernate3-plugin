
Hibernate3 support
------------------

Plugin page: [http://artifacts.griffon-framework.org/plugin/hibernate3](http://artifacts.griffon-framework.org/plugin/hibernate3)


The Hibernate3 plugin enables lightweight access to datasources using [Hibernate3][1].
This plugin does NOT provide domain classes nor dynamic finders like GORM does.

Usage
-----
Upon installation the plugin will generate the following artifacts in
`$appdir/griffon-app/conf`:

 * DataSource.groovy - contains the datasource and pool definitions. Its format
   is equal to GORM's requirements.
   Additional configuration for this artifact is explained in the [datasource][2] plugin.
 * Hibernate3Config.groovy - contains SessionFactory definitions.
 * BootstrapHibernate3.groovy - defines init/destroy hooks for data to be manipulated
   during app startup/shutdown.

A new dynamic method named `withHibernate3` will be injected into all controllers,
giving you access to a `org.hibernate.Session` object, with which you'll be able to
make calls to the database. Remember to make all database calls off the UI thread
otherwise your application may appear unresponsive when doing long computations
inside the UI thread.

This method is aware of multiple databases. If no databaseName is specified
when calling it then the default database will be selected. Here are two example
usages, the first queries against the default database while the second queries
a database whose name has been configured as 'internal'

    package sample
    class SampleController {
        def queryAllDatabases = {
            withHibernate3 { databaseName, session -> ... }
            withHibernate3('internal') { databaseName, session -> ... }
        }
    }

The following list enumerates all the variants of the injected method

 * `<R> R withHibernate3(Closure<R> stmts)`
 * `<R> R withHibernate3(CallableWithArgs<R> stmts)`
 * `<R> R withHibernate3(String databaseName, Closure<R> stmts)`
 * `<R> R withHibernate3(String databaseName, CallableWithArgs<R> stmts)`

These methods are also accessible to any component through the singleton
`griffon.plugins.hibernate3.Hibernate3Enhancer`. You can inject these methods to
non-artifacts via metaclasses. Simply grab hold of a particular metaclass and
call `Hibernate3Enhancer.enhance(metaClassInstance)`.

This plugin relies on the facilities exposed by the [datasource][2] plugin.

Configuration
-------------
### Mapping Files

The plugin expects to find mapping files that conform to the standard class
name patern supported by Hibernate. For example, a class named `sample.Person`
must have a companion mapping file `sample/Person.hbm.xml`. These mapping files
must be placed under `griffon-app/resources` in order to be picked up
automatically by the plugin.

### Hibernate3Aware AST Transformation

The preferred way to mark a class for method injection is by annotating it with
`@griffon.plugins.hibernate3.Hibernate3Aware`. This transformation injects the
`griffon.plugins.hibernate3.Hibernate3ContributionHandler` interface and default
behavior that fulfills the contract.

### Dynamic Method Injection

Dynamic methods will be added to controllers by default. You can
change this setting by adding a configuration flag in `griffon-app/conf/Config.groovy`

    griffon.hibernate3.injectInto = ['controller', 'service']

Dynamic method injection will be skipped for classes implementing
`griffon.plugins.hibernate3.Hibernate3ContributionHandler`.

### Events

The following events will be triggered by this addon

 * Hibernate3ConnectStart[config, dataSourceName] - triggered before connecting to the database
 * Hibernate3ConfigurationAvailable[configuration, dataSourceName, dataSourceConfig, hibernateConfig] - triggered before opening the SessionFactory
 * Hibernate3SessionFactoryCreated[config, dataSourceName, sesstionFactory] - triggered after the SessionFactory was created
 * Hibernate3ConnectEnd[dataSourceName, sessionFactory] - triggered after connecting to the database
 * Hibernate3DisconnectStart[config, dataSourceName, sessionFactory] - triggered before disconnecting from the database
 * Hibernate3DisconnectEnd[config, dataSourceName] - triggered after disconnecting from the database

### Multiple Session Factories

The config file `Hibernate3Config.groovy` defines a default sessionFactory block.
As the name implies this is the SessionFactory used by default, however you can
configure named session factories by adding a new config block. For example
connecting to a database whose name is 'internal' can be done in this way

    sessionFactories {
        internal {
        }
    }

The name of the seesion factory must match the name of a configured dataSource in
`DataSource.groovy`. This block can be used inside the `environments()` block in
the same way as the default sessionFactory block is used.

### Configuration Storage

The plugin will load and store the contents of `Hibernate3Config.groovy` inside the
application's configuration, under the `pluginConfig` namespace. You may retrieve
and/or update values using

    app.config.pluginConfig.hibernate3

### Connect at Startup

The plugin will attempt a connection to the default database at startup. If this
behavior is not desired then specify the following configuration flag in
`Config.groovy`

    griffon.hibernate3.connect.onstartup = false

### Example

A trivial sample application can be found at [https://github.com/aalmiray/griffon_sample_apps/tree/master/persistence/hibernate3][3]

Testing
-------

Dynamic methods will not be automatically injected during unit testing, because
addons are simply not initialized for this kind of tests. However you can use
`Hibernate3Enhancer.enhance(metaClassInstance, hibernate3ProviderInstance)` where
`hibernate3ProviderInstance` is of type `griffon.plugins.hibernate3.Hibernate3Provider`.
The contract for this interface looks like this

    public interface Hibernate3Provider {
        <R> R withHibernate3(Closure<R> closure);
        <R> R withHibernate3(CallableWithArgs<R> callable);
        <R> R withHibernate3(String databaseName, Closure<R> closure);
        <R> R withHibernate3(String databaseName, CallableWithArgs<R> callable);
    }

It's up to you define how these methods need to be implemented for your tests.
For example, here's an implementation that never fails regardless of the
arguments it receives

    class MyHibernate3Provider implements Hibernate3Provider {
        public <R> R withHibernate3(Closure<R> closure) { null }
        public <R> R withHibernate3(CallableWithArgs<R> callable) { null }
        public <R> R withHibernate3(String databaseName, Closure<R> closure) { null }
        public <R> R withHibernate3(String databaseName, CallableWithArgs<R> callable) { null }
    }

This implementation may be used in the following way

    class MyServiceTests extends GriffonUnitTestCase {
        void testSmokeAndMirrors() {
            MyService service = new MyService()
            Hibernate3Enhancer.enhance(service.metaClass, new MyHibernate3Provider())
            // exercise service methods
        }
    }

On the other hand, if the service is annotated with `@Hibernate3Aware` then usage
of `Hibernate3Enhancer` should be avoided at all costs. Simply set
`hibernate3ProviderInstance` on the service instance directly, like so, first the
service definition

    @griffon.plugins.hibernate3.Hibernate3Aware
    class MyService {
        def serviceMethod() { ... }
    }

Next is the test

    class MyServiceTests extends GriffonUnitTestCase {
        void testSmokeAndMirrors() {
            MyService service = new MyService()
            service.hibernate3Provider = new MyHibernate3Provider()
            // exercise service methods
        }
    }

Tool Support
------------

### DSL Descriptors

This plugin provides DSL descriptors for Intellij IDEA and Eclipse (provided
you have the Groovy Eclipse plugin installed). These descriptors are found
inside the `griffon-hibernate3-compile-x.y.z.jar`, with locations

 * dsdl/hibernate3.dsld
 * gdsl/hibernate3.gdsl

### Lombok Support

Rewriting Java AST in a similar fashion to Groovy AST transformations is
possible thanks to the [lombok][4] plugin.

#### JavaC

Support for this compiler is provided out-of-the-box by the command line tools.
There's no additional configuration required.

#### Eclipse

Follow the steps found in the [Lombok][4] plugin for setting up Eclipse up to
number 5.

 6. Go to the path where the `lombok.jar` was copied. This path is either found
    inside the Eclipse installation directory or in your local settings. Copy
    the following file from the project's working directory

         $ cp $USER_HOME/.griffon/<version>/projects/<project>/plugins/hibernate3-<version>/dist/griffon-hibernate3-compile-<version>.jar .

 6. Edit the launch script for Eclipse and tweak the boothclasspath entry so
    that includes the file you just copied

        -Xbootclasspath/a:lombok.jar:lombok-pg-<version>.jar:        griffon-lombok-compile-<version>.jar:griffon-hibernate3-compile-<version>.jar

 7. Launch Eclipse once more. Eclipse should be able to provide content assist
    for Java classes annotated with `@Hibernate3Aware`.

#### NetBeans

Follow the instructions found in [Annotation Processors Support in the NetBeans
IDE, Part I: Using Project Lombok][5]. You may need to specify
`lombok.core.AnnotationProcessor` in the list of Annotation Processors.

NetBeans should be able to provide code suggestions on Java classes annotated
with `@Hibernate3Aware`.

#### Intellij IDEA

Follow the steps found in the [Lombok][4] plugin for setting up Intellij IDEA
up to number 5.

 6. Copy `griffon-hibernate3-compile-<version>.jar` to the `lib` directory

         $ pwd
           $USER_HOME/Library/Application Support/IntelliJIdea11/lombok-plugin
         $ cp $USER_HOME/.griffon/<version>/projects/<project>/plugins/hibernate3-<version>/dist/griffon-hibernate3-compile-<version>.jar lib

 7. Launch IntelliJ IDEA once more. Code completion should work now for Java
    classes annotated with `@Hibernate3Aware`.


[1]: http://hibernate.org/
[2]: /plugin/datasource
[3]: https://github.com/aalmiray/griffon_sample_apps/tree/master/persistence/hibernate3
[4]: /plugin/lombok
[5]: http://netbeans.org/kb/docs/java/annotations-lombok.html

### Building

This project requires all of its dependencies be available from maven compatible repositories.
Some of these dependencies have not been pushed to the Maven Central Repository, however you
can obtain them from [lombok-dev-deps][lombok-dev-deps].

Follow the instructions found there to install the required dependencies into your local Maven
repository before attempting to build this plugin.

[lombok-dev-deps]: https://github.com/aalmiray/lombok-dev-deps