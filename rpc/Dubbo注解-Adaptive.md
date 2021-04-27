### Dubbo注解-Adaptive

1、顶级接口，作为SPI机制的拓展接口，注意要加上@spi注解。

```java
@SPI("dubbo")
public interface AdaptiveExt2 {
    @Adaptive
    String echo(String msg, URL url);
}
```

2、上面接口的三个实现类

```java
public class DubboAdaptiveExt2 implements AdaptiveExt2{

    @Override
    public String echo(String msg, URL url) {
        return "dubbo";
    }
}
```

```java
public class SpringCloudAdaptiveExt2 implements AdaptiveExt2 {

    public String echo(String msg, URL url) {
        return "spring cloud";
    }

}
```

```java
public class ThriftAdaptiveExt2 implements AdaptiveExt2 {

    public String echo(String msg, URL url) {
        return "thrift";
    }
}
```

3、在Resource目录下，添加/META-INF/dubbo/internal/com.yy.adaptive.AdaptiveExt2文件，里面的内容

```properties
dubbo=com.yy.adaptive.DubboAdaptiveExt2
cloud=com.yy.adaptive.SpringCloudAdaptiveExt2
thrift=com.yy.adaptive.ThriftAdaptiveExt2
```

![](../images/rpc/24.png)

### 测试

1、SPI注解中有value，指定默认的实现。

```java
@Test
public void test(){
    AdaptiveExt2 adaptiveExt2 = ExtensionLoader.getExtensionLoader(AdaptiveExt2.class).getAdaptiveExtension();
    System.out.println(adaptiveExt2.echo("test", URL.valueOf("test://localhost/test")));

}
```

执行结果：dubbo。

2、url参数中带cloud

```
@Test
public void test2(){
    AdaptiveExt2 adaptiveExt2 = ExtensionLoader.getExtensionLoader(AdaptiveExt2.class).getAdaptiveExtension();
    System.out.println(adaptiveExt2.echo("test", URL.valueOf("test://localhost/test?adaptive.ext2=cloud")));

}
```

执行结果：spring cloud。

3、类上带有Adaptive注解

```java
@Adaptive
public class ThriftAdaptiveExt2 implements AdaptiveExt2 {

    public String echo(String msg, URL url) {
        return "thrift";
    }
}
@Test
public void test2(){
   AdaptiveExt2 adaptiveExt2 = ExtensionLoader.getExtensionLoader(AdaptiveExt2.class).getAdaptiveExtension();
        System.out.println(adaptiveExt2.echo("test", URL.valueOf("test://localhost/test?adaptive.ext2=cloud")));

}
```

执行结果：thrift。

4、方法上有Adaptive注解，并且value与url中的key相等

```java
@SPI("dubbo")
public interface AdaptiveExt2 {
    @Adaptive({"t"})
    String echo(String msg, URL url);
}
```

```java
@Test
public void test3(){
    AdaptiveExt2 adaptiveExt2 = ExtensionLoader.getExtensionLoader(AdaptiveExt2.class).getAdaptiveExtension();
    System.out.println(adaptiveExt2.echo("test", URL.valueOf("test://localhost/test?t=thrift")));

}
```

执行结果：thrift。

结论：

从上面的几个测试用例，可以得到下面的结论：1. 在类上加上@Adaptive注解的类，是最为明确的创建对应类型Adaptive类。所以他优先级最高。2. @SPI注解中的value是默认值，如果通过URL获取不到关于取哪个类作为Adaptive类的话，就使用这个默认值，当然如果URL中可以获取到，就用URL中的。3. 可以再方法上增加@Adaptive注解，注解中的value与链接中的参数的key一致，链接中的key对应的value就是spi中的name,获取相应的实现类。