# Dubbo 源码分析 - 自适应拓展原理

## 1.原理

我在上一篇文章中分析了 Dubbo 的 SPI 机制，Dubbo SPI 是 Dubbo  框架的核心。Dubbo 中的很多拓展都是通过 SPI 机制进行加载的，比如 Protocol、Cluster、LoadBalance  等。有时，有些拓展并非想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时参数进行加载。这听起来有些矛盾。拓展未被加载，那么拓展方法就无法被调用（静态方法除外）。拓展方法未被调用，就无法进行加载，这似乎是个死结。不过好在也有相应的解决办法，通过代理模式就可以解决这个问题，这里我们将具有代理功能的拓展称之为自适应拓展。Dubbo 并未直接通过代理模式实现自适应拓展，而是代理代理模式基础上，封装了一个更炫的实现方式。Dubbo  首先会为拓展接口生成具有代理功能的代码，然后通过 javassist 或 jdk 编译这段代码，得到 Class  类，最后在通过反射创建代理类。整个过程比较复杂、炫丽。如此复杂的过程最终的目的是为拓展生成代理对象，但实际上每个代理对象的代理逻辑基本一致，均是从 URL  中获取欲加载实现类的名称。因此，我们完全可以把代理逻辑抽出来，并通过动态代理的方式实现自适应拓展。这样做的好处显而易见，方便维护，也方便源码学习者学习和调试代码。本文将在随后实现一个动态代理版的自适应拓展，有兴趣的同学可以继续往下读。

接下来，我们通过一个示例演示自适应拓展类。这个示例取自 Dubbo 官方文档，我这里进行了一定的拓展。这是一个与汽车相关的例子，我们有一个车轮制造厂接口 WheelMaker：

```java
public interface WheelMaker {
    Wheel makeWheel(URL url);
}
```

WheelMaker 接口的 Adaptive 实现类如下：

```java
public class AdaptiveWheelMaker implements WheelMaker {
    public Wheel makeWheel(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        
    	// 1.从 URL 中获取 WheelMaker 名称
        String wheelMakerName = url.getParameter("Wheel.maker");
        if (name == null) {
            throw new IllegalArgumentException("wheelMakerName == null");
        }
        
        // 2.通过 SPI 加载具体的 WheelMaker
        WheelMaker wheelMaker = ExtensionLoader
            .getExtensionLoader(WheelMaker.class).getExtension(wheelMakerName);
        
        // 3.调用目标方法
        return wheelMaker.makeWheel(URL url);
    }
}
```

AdaptiveWheelMaker 是一个代理类，它主要做了三件事情：

1. 从 URL 中获取 WheelMaker 名称
2. 通过 SPI 加载具体的 WheelMaker
3. 调用目标方法

接下来，我们来看看汽车制造厂 CarMaker 接口与其实现类。

```java
public interface CarMaker {
    Car makeCar(URL url);
}

public class RaceCarMaker implements CarMaker {
    WheelMaker wheelMaker;
 
    // 通过 setter 注入 AdaptiveWheelMaker
    public setWheelMaker(WheelMaker wheelMaker) {
        this.wheelMaker = wheelMaker;
    }
 
    public Car makeCar(URL url) {
        Wheel wheel = wheelMaker.makeWheel(url);
        return new RaceCar(wheel, ...);
    }
}
```

RaceCarMaker 持有一个 WheelMaker 类型从成员变量，在程序启动时，我们可以将 AdaptiveWheelMaker 通过 setter 方法注入到 RaceCarMaker 中。在运行时，假设有这样一个 URL 类型的参数：

```
dubbo://192.168.0.101:20880/XxxService?wheel.maker=MichelinWheelMaker
```

RaceCarMaker 的 makeCar 方法将上面的 url 作为参数传给 AdaptiveWheelMaker 的  makeWheel 方法，makeWheel 方法从 url 中提取 wheel.maker 参数，得到  MichelinWheelMaker。之后再通过 SPI 加载名为 MichelinWheelMaker 的实现类，得到具体的  WheelMaker 实例。

上面这个示例展示了自适应拓展类的核心实现 – 在组件方法被调用时，通过代理的方式加载指定的实现类，并调用被代理的方法。

经过以上说明，大家应该搞懂了自适应拓展的原理。接下来，我们深入到源码中，探索自适应拓展生成的过程。

## 2.源码分析

在对自适应拓展生成过程进行深入分析之前，我们先来看一下与自适应拓展息息相关的一个注解，即 Adaptive 注解。该注解的定义如下：

```java
@Documented
@Retention({RetentionPolicy.RUNTIME})
@Target({ElementType.TYPE,ElementType.METHOD})
public @interface Adaptive  {
    String[] value() default {};
}
```

从上面的代码中可知，Adaptive 可注解在类或方法上。注解在类上时，Dubbo 不会为该类生成代理类。注解上方法（接口方法）上时，Dubbo 会为该方法生成代理逻辑。Adaptive 注解在类上的情况很少，在 Dubbo 中，仅有两个类被 Adaptive 注解了，分别是  AdaptiveCompiler 和  AdaptiveExtensionFactory。此种情况表示拓展的加载逻辑由人工编码完成。更多时候，Adaptive  是注解在接口方法上的，表示拓展的加载逻辑需由框架自动生成。Adaptive  注解的地方不同，相应的处理逻辑也是不同的。注解在类上时，处理逻辑比较简单，本文就不分析了。注解在接口方法上时，处理逻辑较为复杂，本章将会重点分析此块逻辑。接下来，我们从 getAdaptiveExtension 方法进行分析。代码如下：

### 2.1 获取自适应拓展

```java
public T getAdaptiveExtension() {
    // 从缓存中获取自适应拓展
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {    // 缓存未命中
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        // 创建自适应拓展
                        instance = createAdaptiveExtension();
                        // 设置拓展到缓存中
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("...");
                    }
                }
            }
        } else {
            throw new IllegalStateException("...");
        }
    }

    return (T) instance;
}
```

getAdaptiveExtension 方法首先会检查缓存，缓存未命中，则调用 createAdaptiveExtension 方法创建自适应拓展。下面，我们看一下 createAdaptiveExtension 方法的代码。

```java
private T createAdaptiveExtension() {
    try {
        // 获取自适应拓展类，并通过反射实例化
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("...");
    }
}
```

createAdaptiveExtension 方法代码比较少，但却包含了三个动作，分别如下：

1. 调用 getAdaptiveExtensionClass 方法获取自适应拓展 Class 对象
2. 通过反射进行实例化
3. 调用 injectExtension 方法向拓展实例中注入依赖

前两个动作比较好理解，第三个动作不好理解，这里简单说明一下。injectExtension 方法通过 setter  方法向目标对象中注入依赖，可以看做是一个简单 IOC 的实现。前面说过，Dubbo  中有两种类型的自适应拓展，一种是手工编码的，一种是自动生成的。手工编码的 Adaptive 拓展中可能存在着一些依赖，而自动生成的  Adaptive 拓展则不会依赖其他类。这里调用 injectExtension  方法的目的是为手工编码的自适应拓展注入依赖，这一点需要大家注意一下。关于 injectExtension 方法，我在[上一篇文章](https://www.tianxiaobo.com/2018/10/13/Dubbo-源码分析-自适应拓展原理/)中已经分析过了，这里不再赘述。接下来，分析 getAdaptiveExtensionClass 方法的逻辑。

```java
private Class<?> getAdaptiveExtensionClass() {
    // 通过 SPI 获取所有的拓展类
    getExtensionClasses();
    // 检查缓存，若缓存不为空，则直接返回缓存
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    // 创建自适应拓展类
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

getAdaptiveExtensionClass 方法也包含了三个步骤，如下：

1. 调用 getExtensionClasses 获取所有的拓展类
2. 检查缓存，若缓存不为空，则返回缓存
3. 若缓存为空，则调用 createAdaptiveExtensionClass 创建自适应拓展类

这三个步骤看起来平淡无奇，似乎没有多讲的必要。但是这些平淡无奇的代码中隐藏了一些细节，需要说明一下。首先从第一个步骤说起，getExtensionClasses 这个方法用于获取某个接口的所有实现类。比如该方法可以获取 Protocol 接口的  DubboProtocol、HttpProtocol、InjvmProtocol 等实现类。在获取实现类的过程中，如果某个某个实现类被  Adaptive 注解修饰了，那么该类就会被赋值给 cachedAdaptiveClass  变量。此时，上面步骤中的第二步条件成立（缓存不为空），直接返回 cachedAdaptiveClass 即可。如果所有的实现类均未被  Adaptive 注解修饰，那么执行第三步逻辑，创建自适应拓展类。相关代码如下：

```java
private Class<?> createAdaptiveExtensionClass() {
    // 构建自适应拓展代码
    String code = createAdaptiveExtensionClassCode();
    ClassLoader classLoader = findClassLoader();
    // 获取编译器实现类
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    // 编译代码，生成 Class
    return compiler.compile(code, classLoader);
}
```

createAdaptiveExtensionClass 方法用于生成自适应拓展类，该方法首先会生成自适应拓展类的源码，然后通过  Compiler 实例（Dubbo 默认使用 javassist 作为编译器）编译源码，得到代理类 Class  实例。接下来，我将重点分析代理类代码生成逻辑。至于代码编译的过程，并非本文范畴，这里就不分析了，大家有兴趣可以自己看看。下面，我们把目光聚焦在  createAdaptiveExtensionClassCode 方法上。

### 2.2 自适应拓展类代码生成

createAdaptiveExtensionClassCode 方法代码略多，约有两百行代码。因此在本节中，我将会对该方法的代码进行拆分分析，以帮助大家更好的理解代码含义。

#### 2.2.1 Adaptive 注解检测

在生成代理类源码之前，createAdaptiveExtensionClassCode 方法首先会通过反射检测接口方法是否包含 Adaptive 注解。对于要生成自适应拓展的接口，Dubbo 要求该接口至少有一个方法被  Adaptive 注解修饰。若不满足此条件，就会抛出运行时异常。相关代码如下：

```java
// 通过反射获取所有的方法
Method[] methods = type.getMethods();
boolean hasAdaptiveAnnotation = false;
// 遍历方法列表
for (Method m : methods) {
    // 检测方法上是否有 Adaptive 注解
    if (m.isAnnotationPresent(Adaptive.class)) {
        hasAdaptiveAnnotation = true;
        break;
    }
}

if (!hasAdaptiveAnnotation)
    // 若所有的方法上均无 Adaptive 注解，则抛出异常
    throw new IllegalStateException("...");
```

#### 2.2.2 生成类

通过 Adaptive 注解检测后，即可开始生成代码。代码生成的顺序与 Java 文件内容顺序一致，首先会生成 package 语句，然后生成 import 语句，紧接着生成类名等代码。整个逻辑如下：

```java
// 生成 package 代码：package + type 所在包
codeBuilder.append("package ").append(type.getPackage().getName()).append(";");
// 生成 import 代码：import + ExtensionLoader 全限定名
codeBuilder.append("\nimport ").append(ExtensionLoader.class.getName()).append(";");
// 生成类代码：public class + type简单名称 + $Adaptive + implements + type全限定名 + {
codeBuilder.append("\npublic class ")
    .append(type.getSimpleName())
    .append("$Adaptive")
    .append(" implements ")
    .append(type.getCanonicalName())
    .append(" {");

// ${生成方法}

codeBuilder.append("\n}");
```

这里，我用 ${…} 占位符代表其他代码的生成逻辑，该部分逻辑我将在随后进行分析。上面代码不是很难理解，这里我直接通过一个例子展示该段代码所生成的内容。以 Dubbo 的 Protocol 接口为例，生成的代码如下：

```java
package com.alibaba.dubbo.rpc;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
    // 省略方法代码
}
```

#### 2.2.3 生成方法

一个方法可以被 Adaptive 注解修饰，也可以不被修饰。这里将未被 Adaptive 注解修饰的方法称为“无 Adaptive 注解方法”，下面我们先来看看此种方法的代码生成逻辑是怎样的。

#####  2.2.3.1 无 Adaptive 注解方法代码生成

对于接口方法，我们可以按照需求标注 Adaptive 注解。以 Protocol 接口为例，该接口的 destroy 和 getDefaultPort 未标注 Adaptive  注解，其他方法均标注了 Adaptive 注解。Dubbo 不会为没有标注 Adaptive  注解的方法生成代理逻辑，对于该种类型的方法，仅会生成一句抛出异常的代码。生成逻辑如下：

```java
for (Method method : methods) {
    
    // 省略无关逻辑

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    // 如果方法上无 Adaptive 注解，则生成 throw new UnsupportedOperationException(...) 代码
    if (adaptiveAnnotation == null) {
        // 生成规则：
        // throw new UnsupportedOperationException(
        //     "method " + 方法签名 + of interface + 全限定接口名 + is not adaptive method!”)
        code.append("throw new UnsupportedOperationException(\"method ")
            .append(method.toString()).append(" of interface ")
            .append(type.getName()).append(" is not adaptive method!\");");
    } else {
        // 省略无关逻辑
    }
    
    // 省略无关逻辑
}
```

以 Protocol 接口的 destroy 方法为例，上面代码生成的内容如下：

```java
throw new UnsupportedOperationException(
            "method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
```

##### 2.2.3.2 获取 URL 数据

前面说过方法代理逻辑会从 URL  中提取目标拓展的名称，因此代码生成逻辑的一个重要的任务是从方法的参数列表获取其他参数中获取 URL 数据。举个例子说明一下，我们要为  Protocol 接口的 refer 和 export 方法生成代理逻辑。在运行时，通过反射得到的方法定义大致如下：

```java
Invoker refer(Class<T> arg0, URL arg1) throws RpcException;
Exporter export(Invoker<T> arg0) throws RpcException;
```

对于 refer 方法，通过遍历 refer 的参数列表即可获取 URL 数据，这个还比较简单。对于 export 方法，获取 URL  数据则要麻烦一些。export 参数列表中没有 URL 参数，因此需要从 Invoker 参数中获取 URL 数据。获取方式是调用  Invoker 中可返回 URL 的 getter 方法，比如 getUrl。如果 Invoker 中无相关 getter  方法，此时则会抛出异常。整个逻辑如下：

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // ${无 Adaptive 注解方法代码生成}
    } else {
    	int urlTypeIndex = -1;
        // 遍历参数列表，确定 URL 参数位置
        for (int i = 0; i < pts.length; ++i) {
            if (pts[i].equals(URL.class)) {
                urlTypeIndex = i;
                break;
            }
        }
        if (urlTypeIndex != -1) {    // 参数列表中存在 URL 参数
            // 为 URL 类型参数生成判空代码，格式如下：
            // if (arg + urlTypeIndex == null) 
            //     throw new IllegalArgumentException("url == null");
            String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"url == null\");",
                                     urlTypeIndex);
            code.append(s);

            // 为 URL 类型参数生成赋值代码，即 URL url = arg1 或 arg2，或 argN
            s = String.format("\n%s url = arg%d;", URL.class.getName(), urlTypeIndex);
            code.append(s);
            
        } else {    // 参数列表中不存在 URL 类型参数
            String attribMethod = null;

            LBL_PTS:
            // 遍历方法的参数类型列表
            for (int i = 0; i < pts.length; ++i) {
                // 获取某一类型参数的全部方法
                Method[] ms = pts[i].getMethods();
                // 遍历方法列表，寻找可返回 URL 的 getter 方法
                for (Method m : ms) {
                    String name = m.getName();
                    // 1. 方法名以 get 开头，或方法名大于3个字符
                    // 2. 方法的访问权限为 public
                    // 3. 方法非静态类型
                    // 4. 方法参数数量为0
                    // 5. 方法返回值类型为 URL
                    if ((name.startsWith("get") || name.length() > 3)
                        && Modifier.isPublic(m.getModifiers())
                        && !Modifier.isStatic(m.getModifiers())
                        && m.getParameterTypes().length == 0
                        && m.getReturnType() == URL.class) {
                        urlTypeIndex = i;
                        attribMethod = name;
                        
                        // 结束 for (int i = 0; i < pts.length; ++i) 循环
                        break LBL_PTS;
                    }
                }
            }
            if (attribMethod == null) {
                // 如果所有参数中均不包含可返回 URL 的 getter 方法，则抛出异常
                throw new IllegalStateException("...");
            }

            // 为包含可返回 URL 的参数生成判空代码，格式如下：
            // if (arg + urlTypeIndex == null) 
            //     throw new IllegalArgumentException("参数全限定名 + argument == null");
            String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"%s argument == null\");",
                                     urlTypeIndex, pts[urlTypeIndex].getName());
            code.append(s);

            // 为 getter 方法返回的 URL 生成判空代码，格式如下：
            // if (argN.getter方法名() == null) 
            //     throw new IllegalArgumentException(参数全限定名 + argument getUrl() == null);
            s = String.format("\nif (arg%d.%s() == null) throw new IllegalArgumentException(\"%s argument %s() == null\");",
                              urlTypeIndex, attribMethod, pts[urlTypeIndex].getName(), attribMethod);
            code.append(s);

            // 生成赋值语句，格式如下：
            // URL全限定名 url = argN.getter方法名()，比如 
            // com.alibaba.dubbo.common.URL url = invoker.getUrl();
            s = String.format("%s url = arg%d.%s();", URL.class.getName(), urlTypeIndex, attribMethod);
            code.append(s);
        }
        
        // 省略无关代码
    }
    
    // 省略无关代码
}
```

上面代码有点多，但并不是很难看懂。这段代码主要是为了获取 URL 数据，并为之生成判空和赋值代码。以 Protocol 的 refer 和 export 方法为例，上面代码会为它们生成如下内容（代码已格式化）：

```java
refer:
if (arg1 == null) 
    throw new IllegalArgumentException("url == null");
com.alibaba.dubbo.common.URL url = arg1;

export:
if (arg0 == null) 
    throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
if (arg0.getUrl() == null) 
    throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
com.alibaba.dubbo.common.URL url = arg0.getUrl();
```

##### 2.2.3.3 获取 Adaptive 注解值

Adaptive  注解值 value 类型为 String[]，可填写多个值，默认情况下为空数组。若 value 为非空数组，直接获取数组内容即可。若 value 为空数组，则需进行额外处理。处理的过程是将类名转换为字符数组，然后遍历字符数组，并将字符加入到 StringBuilder  中。若字符为大写字母，则向 StringBuilder 中添加点号，随后将字符变为小写存入 StringBuilder 中。比如  LoadBalance 经过处理后，得到 load.balance。

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // ${无 Adaptive 注解方法代码生成}
    } else {
        // ${获取 URL 数据}
        
        String[] value = adaptiveAnnotation.value();
        // value 为空数组
        if (value.length == 0) {
            // 获取类名，并将类名转换为字符数组
            char[] charArray = type.getSimpleName().toCharArray();
            StringBuilder sb = new StringBuilder(128);
            // 遍历字节数组
            for (int i = 0; i < charArray.length; i++) {
                // 检测当前字符是否为大写字母
                if (Character.isUpperCase(charArray[i])) {
                    if (i != 0) {
                        // 向 sb 中添加点号
                        sb.append(".");
                    }
                    // 将字符变为小写，并添加到 sb 中
                    sb.append(Character.toLowerCase(charArray[i]));
                } else {
                    // 添加字符到 sb 中
                    sb.append(charArray[i]);
                }
            }
            value = new String[]{sb.toString()};
        }
        
        // 省略无关代码
    }
    
    // 省略无关逻辑
}
```

##### 2.2.3.4 检测 Invocation 参数

此段逻辑是检测方法列表中是否存在 Invocation 类型的参数，若存在，则为其生成判空代码和其他一些代码。相应的逻辑如下：

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();    // 获取参数类型列表
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // ${无 Adaptive 注解方法代码生成}
    } else {
        // ${获取 URL 数据}
        
        // ${获取 Adaptive 注解值}
        
        boolean hasInvocation = false;
        // 遍历参数类型列表
        for (int i = 0; i < pts.length; ++i) {
            // 判断当前参数名称是否等于 com.alibaba.dubbo.rpc.Invocation
            if (pts[i].getName().equals("com.alibaba.dubbo.rpc.Invocation")) {
                // 为 Invocation 类型参数生成判空代码
                String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"invocation == null\");", i);
                code.append(s);
                // 生成 getMethodName 方法调用代码，格式为：
                //    String methodName = argN.getMethodName();
                s = String.format("\nString methodName = arg%d.getMethodName();", i);
                code.append(s);
                
                // 设置 hasInvocation 为 true
                hasInvocation = true;
                break;
            }
        }
    }
    
    // 省略无关逻辑
}
```

##### 2.2.3.5 生成拓展名获取逻辑

本段逻辑用于根据 SPI 和 Adaptive 注解值生成“拓展名获取逻辑”，同时生成逻辑也受 Invocation 类型参数影响，综合因素导致本段逻辑相对复杂。本段逻辑可以会生成但不限于下面的代码：

```java
tring extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
```

或

```java
String extName = url.getMethodParameter(methodName, "loadbalance", "random");
```

亦或是:

```java
String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
```

本段逻辑复杂指出在于条件分支比较多，大家在阅读源码时需要知道每个条件分支的意义是什么，否则不太容易看懂相关代码。好了，其他的就不多说了，开始分析本段逻辑。

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // $无 Adaptive 注解方法代码生成}
    } else {
        // ${获取 URL 数据}
        
        // ${获取 Adaptive 注解值}
        
        // ${检测 Invocation 参数}
        
        // 设置默认拓展名，cachedDefaultName = SPI 注解值，比如 Protocol 接口上标注的 
        // SPI 注解值为 dubbo。默认情况下，SPI 注解值为空串，此时 cachedDefaultName = null
        String defaultExtName = cachedDefaultName;
        String getNameCode = null;
        
        // 遍历 value，这里的 value 是 Adaptive 的注解值，2.2.3.3 节分析过 value 变量的获取过程。
        // 此处循环目的是生成从 URL 中获取拓展名的代码，生成的代码会赋值给 getNameCode 变量。注意这
        // 个循环的遍历顺序是由后向前遍历的。
        for (int i = value.length - 1; i >= 0; --i) {
            if (i == value.length - 1) {    // 当 i 为最后一个元素的坐标时
                if (null != defaultExtName) {   // 默认拓展名非空
                    // protocol 是 url 的一部分，可通过 getProtocol 方法获取，其他的则是从
                    // URL 参数中获取。所以这里要判断 value[i] 是否为 protocol
                    if (!"protocol".equals(value[i]))
                    	// hasInvocation 用于标识方法参数列表中是否有 Invocation 类型参数
                        if (hasInvocation)
                            // 生成的代码功能等价于下面的代码：
                            //   url.getMethodParameter(methodName, value[i], defaultExtName)
                            // 以 LoadBalance 接口的 select 方法为例，最终生成的代码如下：
                            //   url.getMethodParameter(methodName, "loadbalance", "random")
                            getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                    	else
                    		// 生成的代码功能等价于下面的代码：
	                        //   url.getParameter(value[i], defaultExtName)
	                        getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                    else
                    	// 生成的代码功能等价于下面的代码：
                        //   ( url.getProtocol() == null ? defaultExtName : url.getProtocol() )
                        getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
                    
                } else {    // 默认拓展名为空
                    if (!"protocol".equals(value[i]))
                        if (hasInvocation)
                        	// 生成代码格式同上
                            getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
	                    else
	                    	// 生成的代码功能等价于下面的代码：
	                        //   url.getParameter(value[i])
	                        getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                    else
                    	// 生成从 url 中获取协议的代码，比如 "dubbo"
                        getNameCode = "url.getProtocol()";
                }
            } else {
                if (!"protocol".equals(value[i]))
                    if (hasInvocation)
                        // 生成代码格式同上
                        getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
	                else
	                	// 生成的代码功能等价于下面的代码：
	                    //   url.getParameter(value[i], getNameCode)
	                    // 以 Transporter 接口的 connect 方法为例，最终生成的代码如下：
	                    //   url.getParameter("client", url.getParameter("transporter", "netty"))
	                    getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                else
                    // 生成的代码功能等价于下面的代码：
                    //   url.getProtocol() == null ? getNameCode : url.getProtocol()
                    // 以 Protocol 接口的 connect 方法为例，最终生成的代码如下：
                    //   url.getProtocol() == null ? "dubbo" : url.getProtocol()
                    getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
            }
        }
        // 生成 extName 赋值代码
        code.append("\nString extName = ").append(getNameCode).append(";");
        // 生成 extName 判空代码
        String s = String.format("\nif(extName == null) " +
                                 "throw new IllegalStateException(\"Fail to get extension(%s) name from url(\" + url.toString() + \") use keys(%s)\");",
                                 type.getName(), Arrays.toString(value));
        code.append(s);
    }
    
    // 省略无关逻辑
}
```

上面代码已经进行了大量的注释，不过看起来任然不是很好理解。既然如此，那么建议大家写点测试代码，对 Protocol、LoadBalance 以及 Transporter 等接口的自适应拓展类代码生成过程进行调试。这里我以 Transporter  接口的自适应拓展类代码生成过程进行分析。首先看一下 Transporter 接口的定义，如下：

```java
@SPI("netty")
public interface Transporter {
	// @Adaptive({server, transporter})  @Adaptive({Constants.SERVER_KEY,Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    // @Adaptive({client, transporter})
 @Adaptive({Constants.CLIENT_KEY,Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
}
```

下面对 connect 方法代理逻辑生成的过程进行分析，此时生成代理逻辑所用到的变量和值如下：

```java
String defaultExtName = "netty";
boolean hasInvocation = false;
String getNameCode = null;
String[] value = ["client", "transporter"];
```

下面对 value 数组进行遍历，此时 i = 1, value[i] = “transporter”，生成的代码如下：

```JAVA
getNameCode = url.getParameter("transporter", "netty");
```

接下来，for 循环继续执行，此时 i = 0, value[i] = “client”，生成的代码如下：

```JAVA
getNameCode = url.getParameter("client", url.getParameter("transporter", "netty"));
```

for 循环结束运行，现在生成 extName 变量及判空代码，如下：

```JAVA
String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
if (extName == null) {
    throw new IllegalStateException(
        "Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString()
        + ") use keys([client, transporter])");
}
```

到此，connect 方法的拓展名获取代码就生成好了。如果大家不是很明白，建议自己调试走一遍。好了，本节先到这里。

##### 2.2.3.6 生成拓展加载与目标方法调用逻辑

上一节的逻辑生成拓展名 extName 获取逻辑，接下来要做的是根据拓展名加载拓展实例，并调用拓展实例的目标方法。相关逻辑如下：

```JAVA
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // $无 Adaptive 注解方法代码生成}
    } else {
        // ${获取 URL 数据}
        
        // ${获取 Adaptive 注解值}
        
        // ${检测 Invocation 参数}
        
        // ${生成拓展名获取逻辑}
        
        // 生成拓展获取代码，格式如下：
        // type全限定名 extension = (type全限定名)ExtensionLoader全限定名
        //     .getExtensionLoader(type全限定名.class).getExtension(extName);
        // Tips: 格式化字符串中的 %<s 表示使用前一个转换符所描述的参数，即 type 全限定名
        s = String.format("\n%s extension = (%<s)%s.getExtensionLoader(%s.class).getExtension(extName);",
                        type.getName(), ExtensionLoader.class.getSimpleName(), type.getName());
        code.append(s);

		// 如果方法有返回值类型非 void，则生成 return 语句。
        if (!rt.equals(void.class)) {
            code.append("\nreturn ");
        }

        // 生成目标方法调用逻辑，格式为：
        //     extension.方法名(arg0, arg2, ..., argN);
        s = String.format("extension.%s(", method.getName());
        code.append(s);
        for (int i = 0; i < pts.length; i++) {
            if (i != 0)
                code.append(", ");
            code.append("arg").append(i);
        }
        code.append(");");   
    }
    
    // 省略无关逻辑
}
```

```JAVA
com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
            .getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
return extension.refer(arg0, arg1)
```

##### 2.2.3.7 生成完整的方法

本节进行代码生成的收尾工作，主要用于生成方法定义的代码。相关逻辑如下:

```JAVA
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // $无 Adaptive 注解方法代码生成}
    } else {
        // ${获取 URL 数据}
        
        // ${获取 Adaptive 注解值}
        
        // ${检测 Invocation 参数}
        
        // ${生成拓展名获取逻辑}
        
        // ${生成拓展加载与目标方法调用逻辑}
    }
}
    
// public + 返回值全限定名 + 方法名 + (
codeBuilder.append("\npublic ")
    .append(rt.getCanonicalName())
    .append(" ")
    .append(method.getName())
    .append("(");

// 添加参数列表代码
for (int i = 0; i < pts.length; i++) {
    if (i > 0) {
        codeBuilder.append(", ");
    }
    codeBuilder.append(pts[i].getCanonicalName());
    codeBuilder.append(" ");
    codeBuilder.append("arg").append(i);
}
codeBuilder.append(")");

// 添加异常抛出代码
if (ets.length > 0) {
    codeBuilder.append(" throws ");
    for (int i = 0; i < ets.length; i++) {
        if (i > 0) {
            codeBuilder.append(", ");
        }
        codeBuilder.append(ets[i].getCanonicalName());
    }
}
codeBuilder.append(" {");
codeBuilder.append(code.toString());
codeBuilder.append("\n}");
```

```JAVA
public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) {
    // 方法体
}
```



https://www.tianxiaobo.com/2018/10/13/Dubbo-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E8%87%AA%E9%80%82%E5%BA%94%E6%8B%93%E5%B1%95%E5%8E%9F%E7%90%86/