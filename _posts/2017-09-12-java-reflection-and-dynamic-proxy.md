---
layout: post
title: Java中的反射和动态代理
tags: [Java]
comment: false
---

最近一直都在学习 Java 安全相关的东西，这两个玩意可以说是经常碰到了，做个笔记。话说要是在两个月前我是万万想不到我现在整天接触的居然是 Java，一边学习 Java 一边搞 = =

## 0x00 反射机制
利用反射机制能干的事情：
- 在运行时动态对任意的类实例化
- 在运行时分析类，获取类的成员变量和方法等信息
- 调用任意方法

## 0x01 获取 Class 对象的三种方法
Class 类保存了所有对象运行时的类型标识，虚拟机在运行时根据类型信息再去选择对应的方法来执行。这里的 Class 可以理解为类类型，跟平时说的 Class 是有区别的。下面是如何去获取一个 Class 类型的实例。

#### Object 类中的 getClass() 方法
```java
Test t = new Test();
Class c = t.getClass();
```

#### 调用静态方法 forName()
```java
String className = "java.util.Random";
Class c = Class.forName(className);
```
只有在 className 是类名或接口名时才能执行，不然 forName() 方法会抛出一个 checked 异常。

#### 任意的 Java 类型的 class 变量
```java
Class c1 = Random.class;
Class c2 = int.class;
```

用 newInstance() 方法便可以返回对应类的新实例。
```java
Test t = new Test();
Class c = t.getClass();
Object obj = c.newInstance();
//需要转换类型才可以使用新实例，不然用 obj 去调用 Test 中的方法会出错
Test obj1 = (Test)c.newInstance();
```

## 0x02 获取类信息的操作
获取类成员变量：
- Field[] getFields()
- Field[] getDeclaredFields()
getFields() 返回包含当前类和父类中 public 的成员变量的一个数组，getDeclaredFields() 返回包含当前类的所有成员变量的一个数组。通过 getName() 和 getType() 方法可以获取到其名字和类型。

获取类方法：
- Method[] getMethods()
- Method[] getDeclareMethods()
效果跟上面的类似，只是 getDeclareMethods() 还能得到接口的全部方法。通过 getName() 和 getReturnType() 方法可以获取到其名字和返回类型。

也可以通过其名字来获取到对应的变量，方法名就是上面那些方法名去掉 s 后传名字作为参数。通过 get() 和 set() 方法可以获取类成员变量的值或者是给类成员变量设置一个值，需要有访问权限，不过也可以通过 setAccessible() 方法来覆盖访问控制。下面一个例子：
```java
import java.lang.reflect.Field;

/**
 * Created by k1n9 on 2017/9/12.
 */
public class ReflectionTest {
    public static void main(String[] args) {
        try {
            Class c = Class.forName("Rtest");
            Object obj = c.newInstance();

            try {
                Field f = c.getDeclaredField("secret");
                f.setAccessible(true);
                f.set(obj, "whoami");
                System.out.println(f.get(obj));
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }
}

class PRtest {
    public String word;
}

class Rtest extends PRtest{
    public String name;
    private String secret;

    public void publicfunc() {
        System.out.println(this.name);
    }

    private void privatefunc() {
        System.out.println(this.secret);
    }
}
```
还有其它一些方法，比如获取构造器的等等。

## 0x03 类方法调用
类方法调用其实跟类成员变量的操作差不多，用到的是 Method 类中的 invoke() 方法，也会有访问权限的问题，但是同样可以通过 setAccessible() 方法来解决。
```java
Method m = c.getDeclaredMethod("privatefunc");
m.setAccessible(true);
m.invoke(obj);
```

## 0x04 动态代理
提供一个代理对象，用代理对象来取代对原对象的访问，这里原对象的类需要时实现接口的类。也有可以实现对不需要实现接口的类的对象的代理，但是需要引入别的库了。

#### 动态代理实现
- 实现调用处理器 InvocationHandler 接口
- 用 newProxyInstance 来获取一个代理对象
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)

newProxyInstance 这个方法中需要用到的类加载器，和接口参数都是可以通过反射来获得的，写的例子：
InterfaceTest.java

```java
public interface InterfaceTest {
    public void action();
}
```

InvocationhandlerTest.java

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/**
 * Created by k1n9 on 2017/9/12.
 */
public class InvocationhandlerTest implements InvocationHandler {
    private Object target;

    public InvocationhandlerTest(Object t) {
        target = t;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return method.invoke(target);
    }
}
```

ProxyTest.java

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

/**
 * Created by k1n9 on 2017/9/12.
 */
public class ProxyTest {
    public static void main(String[] args) {
        Ptest t = new Ptest();
        Class<?> c = t.getClass();
        InvocationHandler h = new InvocationhandlerTest(t);

        InterfaceTest proxyobj = (InterfaceTest) Proxy.newProxyInstance(c.getClassLoader(), c.getInterfaces(), h);
        proxyobj.action();
    }
}

class Ptest implements InterfaceTest {
    public void action() {
        System.out.println("action called!");
    }
}
```
代理对象对方法的调用都是通过调用处理器中的 invoke 方法来调用的，Ysoerial 中的一些 payload 的触发也用到了这一点。