## BeanNameAware

作用：让Bean获取自己在BeanFactory配置中的名字（根据情况是id或者name）。 
  Spring自动调用。并且会在Spring自身完成Bean配置之后，且在调用任何Bean生命周期回调（初始化或者销毁）方法之前就调用这个方法。换言之，在程序中使用BeanFactory.getBean(String beanName)之前，Bean的名字就已经设定好了。

## BeanFactoryAware

作用：让Bean获取配置他们的BeanFactory的引用。

这个方法可能是在根据某个配置文件创建了一个新工厂之后，Spring才调用这个方法，并把BeanFactory注入到Bean中。 
 让bean获取配置自己的工厂之后，当然可以在Bean中使用这个工厂的getBean()方法，但是，实际上非常不推荐这样做，因为结果是进一步加大Bean与Spring的耦合，而且，能通过DI注入进来的尽量通过DI来注入。 
 当然，除了查找bean，BeanFactory可以提供大量其他的功能，例如销毁singleton模式的Bean。 
 factory.preInstantiateSingletons();方法。preInstantiateSingletons()方法立即实例化所有的Bean实例，有必要对这个方法和Spring加载bean的机制做个简单说明。 
  方法本身的目的是让Spring立即处理工厂中所有Bean的定义，并且将这些Bean全部实例化。因为Spring默认实例化Bean的情况下，采用的是lazy机制，换言之，如果不通过getBean()方法（BeanFactory或者ApplicationContext的方法）获取Bean的话，那么为了节省内存将不实例话Bean，只有在Bean被调用的时候才实例化他们