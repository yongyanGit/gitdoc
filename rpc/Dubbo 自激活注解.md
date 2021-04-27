### Dubbo 自激活注解

1、定义一个顶级接口

```java
@SPI
public interface ActivateExt1 {
    String echo(String msg);
}
```

2、接口的五个实现类

```java
@Activate(group = {"default_group"})
public class ActivateExt1Impl implements ActivateExt1{

    @Override
    public String echo(String msg) {
        return msg;
    }
}
```

```java
@Activate(group = {"group1","group2"})
public class GroupActivateExtImpl implements ActivateExt1{

    @Override
    public String echo(String msg) {
        return msg;
    }
}
```

```java
@Activate(group = {"order"},order = 1)
public class OrderActivateExtImpl1 implements ActivateExt1{

    @Override
    public String echo(String msg) {
        return msg;
    }
}
```

```java
@Activate(group = {"order"},order = 2)
public class OrderActivateExtImpl2 implements ActivateExt1{

    @Override
    public String echo(String msg) {
        return msg;
    }
}
```

```java
@Activate(value = {"value1"}, group = {"value"})
public class ValueActivateActivateExtImpl implements ActivateExt1{

    @Override
    public String echo(String msg) {
        return msg;
    }
}
```

在Resource目录下，添加/META-INF/dubbo/internal/com.yy.activate.ActivateExt1文件，里面的内容

```properties
group=com.yy.activate.GroupActivateExtImpl
value=com.yy.activate.ValueActivateActivateExtImpl
order1=com.yy.activate.OrderActivateExtImpl1
order2=com.yy.activate.OrderActivateExtImpl2
default_group=com.yy.activate.ActivateExt1Impl
```

测试一：只传入group，筛选出group为default_group的拓展类：

```java
public void test4(){
    ExtensionLoader<ActivateExt1> loader = ExtensionLoader.getExtensionLoader(ActivateExt1.class);
    URL url = URL.valueOf("test://localhost/test");
    //查询组为default_group的ActivateExt1的实现
    List<ActivateExt1> list = loader.getActivateExtension(url, new String[]{}, "default_group");
    System.out.println(list.size());
    System.out.println(list.get(0).getClass());
}
```

执行结果：

```
1
class com.yy.activate.ActivateExt1Impl
```

测试二：传入url参数与group，会先根据group进行筛选再根据参数来判断

```java
@Test
public void test5(){

    ExtensionLoader<ActivateExt1> loader = ExtensionLoader.getExtensionLoader(ActivateExt1.class);
    URL url = URL.valueOf("test://localhost/test");
    url = url.addParameter("value1","value");
  
    List<ActivateExt1> list = loader.getActivateExtension(url, new String[]{}, "value");
    System.out.println(list.size());
    System.out.println(list.get(0).getClass());

}
```