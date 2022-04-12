# 前言

Java里面的的单例模式分为

- 饿汉式
- 懒汉式
- DCL
- 静态内部类
- 枚举单例

前面的几个实现方式都不是本文的重点，重点是记录一下为什么枚举单例是最优秀的单例实现方式



# 单例实现要解决的几个点

- 

# 各模式代码及优缺点

```java
/**
 * 饿汉式
 * 优点：简单快捷
 * 缺点：不管用不用都初始化
 */
class HungerSingle {
    private static HungerSingle instance = new HungerSingle();
    private HungerSingle() {

    }

    public static HungerSingle getInstance() {
        return instance;
    }
}

/**
 * 解决饿汉式预初始化的缺点
 * 缺点：锁粒度大，性能低
 */
class LazySingle {
    private static LazySingle instance;

    private LazySingle() {

    }

    public static synchronized LazySingle getInstance() {
        if (instance == null) {
            instance = new LazySingle();
        }
        return instance;
    }
}

/**
 * DCL 单例
 * 对比懒汉式，提升了锁性能
 */
class DCLSingle {
    /**
     禁止重排序，保证变量线程间可见
     * 重排序情况：o = new o() 会被拆分称三个指令，
     * 1. 分配空间
     * 2。实例化
     * 3.将变量指向内存空间
     * 上述三个指令有可能是同时执行的，这个情况会出现instance 是一个只指向了内存地址，但是实际上并没有实例化的空对象
     */
    private static volatile DCLSingle instance;
    private DCLSingle() {
    }
    public static DCLSingle getInstance() {
        if (instance == null) {
            synchronized (DCLSingle.class) {
                if (instance == null) {
                    //第一次判空，是存在并发的
                    instance = new DCLSingle();
                }
            }
        }
        return instance;
    }
}

/**
 * 静态内部类形式
 * 1.静态内部类，在调用到的时候才会去创建实例
 * 2。
 *
 * 类初始化的几个时机
 * 1、new 或者调用静态方法
 * 2、初始化子类的时候，会去初始化父类
 * 3、反射调用类的时候
 * 4、主类启动
 *
 * 静态内部类的线程安全交给JVM进行保证，看似已经是最好的方式了，
 * 至今未解决的缺点 ： 但是还有一个就是反射修改和序列化方式的方式新建
 */
class StaticInnerSingle {
    private StaticInnerSingle() {
    }
    private static class Inner {
        private static StaticInnerSingle instance = new StaticInnerSingle();
    }
    public static StaticInnerSingle getInstance() {
        return Inner.instance;
    }


}
/**
 * 最佳单例实现方式，解决了线程问题、序列化、反射问题
 * 优点
 * 1、代码简单
 * 2、解决了反射创建
 *      因为反射里面的newinstance()里面写死了如果是枚举类则反射创建不允许 java.lang.reflect.Constructor#newInstance(java.lang.Object...)
 * 3、序列化问题解决
 *      JVM特殊处理，在调用readObject的时候 回去堆中找枚举对象，不会新建
 *
 */
enum EnumSingle implements Serializable{
    /**
     * 单例
     */
    INSTANCE;

    public EnumSingle getInstance(){
        return INSTANCE;
    }
}
```





# 思考

## 为什么枚举单例是最佳实现

### 解决了反射生成对象

在前四种非枚举的，都会存在一个问题如下

```java
        StaticInnerSingle instance = StaticInnerSingle.getInstance();
        Constructor<StaticInnerSingle> declaredConstructor = 	            StaticInnerSingle.class.getDeclaredConstructor();
        declaredConstructor.setAccessible(true);
        StaticInnerSingle staticInnerSingle = declaredConstructor.newInstance();
        System.out.println(staticInnerSingle == instance);
```

这样的反射`newinstance()`生成对象

声明为enum实际是也会继承`java.lang.Enum`抽象类，可以看到这个类只有一个有参构造方法，所以在反射获取构造方法的时

```java
    public static void main(String[] args) throws IllegalAccessException, InvocationTargetException, InstantiationException, NoSuchMethodException {
        EnumSingleton singleton1=EnumSingleton.INSTANCE;
        EnumSingleton singleton2=EnumSingleton.INSTANCE;
        System.out.println("正常情况下，实例化两个实例是否相同："+(singleton1==singleton2));
        Constructor<EnumSingleton> constructor= null;
//        constructor = EnumSingleton.class.getDeclaredConstructor();
        constructor = EnumSingleton.class.getDeclaredConstructor(String.class,int.class);//其父类的构造器
        constructor.setAccessible(true);
        EnumSingleton singleton3= null;
        //singleton3 = constructor.newInstance();
        singleton3 = constructor.newInstance("testInstance",66);
        System.out.println(singleton1+"\n"+singleton2+"\n"+singleton3);
        System.out.println("通过反射攻击单例模式情况下，实例化两个实例是否相同："+(singleton1==singleton3));
    }
```

这个结果是抛出错误

```java
Exception in thread "main" java.lang.IllegalArgumentException: Cannot reflectively create enum objects
```

跟踪代码`java.lang.reflect.Constructor#newInstance#line416`进去可以看到

```java
    @CallerSensitive
    public T newInstance(Object ... initargs)
        throws InstantiationException, IllegalAccessException,
               IllegalArgumentException, InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, null, modifiers);
            }
        }
                   //这里
        if ((clazz.getModifiers() & Modifier.ENUM) != 0)
            throw new IllegalArgumentException("Cannot reflectively create enum objects");
        ConstructorAccessor ca = constructorAccessor;   // read volatile
        if (ca == null) {
            ca = acquireConstructorAccessor();
        }
        @SuppressWarnings("unchecked")
        T inst = (T) ca.newInstance(initargs);
        return inst;
    }
```

结果就是 JDK为enum规定了反射的不可创建

### 解决了序列化的问题

```java
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
//        StaticInnerSingle instance = StaticInnerSingle.getInstance();
//        Constructor<StaticInnerSingle> declaredConstructor = StaticInnerSingle.class.getDeclaredConstructor();
//        declaredConstructor.setAccessible(true);
//        StaticInnerSingle staticInnerSingle = declaredConstructor.newInstance();
//        System.out.println(staticInnerSingle == instance);
        EnumSingle instance = EnumSingle.INSTANCE;
//        Constructor<EnumSingle> declaredConstructor = EnumSingle.class.getDeclaredConstructor();
//        declaredConstructor.setAccessible(true);
//        EnumSingle enumSingle = declaredConstructor.newInstance();
//        System.out.println(instance == enumSingle);

		//这里用了 commons-lang3工具
        byte[] serialize = SerializationUtils.serialize(instance);
        EnumSingle deserialize = SerializationUtils.deserialize(serialize);
        System.out.println(deserialize == instance);
    }
 结果是true
```

跟踪代码可以发现，在反序列化的时候，JDK又对enum做了特殊处理，如果是枚举类的话，会根据类名从堆里面把原有的枚举类load出来当作结果。

调用栈：

![image-20220412190030993](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/image-20220412190030993.png)

```java
java.lang.Enum#valueOf
/**
     * Returns the enum constant of the specified enum type with the
     * specified name.  The name must match exactly an identifier used
     * to declare an enum constant in this type.  (Extraneous whitespace
     * characters are not permitted.)
     *
     * <p>Note that for a particular enum type {@code T}, the
     * implicitly declared {@code public static T valueOf(String)}
     * method on that enum may be used instead of this method to map
     * from a name to the corresponding enum constant.  All the
     * constants of an enum type can be obtained by calling the
     * implicit {@code public static T[] values()} method of that
     * type.
     *
     * @param <T> The enum type whose constant is to be returned
     * @param enumType the {@code Class} object of the enum type from which
     *      to return a constant
     * @param name the name of the constant to return
     * @return the enum constant of the specified enum type with the
     *      specified name
     * @throws IllegalArgumentException if the specified enum type has
     *         no constant with the specified name, or the specified
     *         class object does not represent an enum type
     * @throws NullPointerException if {@code enumType} or {@code name}
     *         is null
     * @since 1.5
     */
    public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                                String name) {
        T result = enumType.enumConstantDirectory().get(name);
        if (result != null)
            return result;
        if (name == null)
            throw new NullPointerException("Name is null");
        throw new IllegalArgumentException(
            "No enum constant " + enumType.getCanonicalName() + "." + name);
    }
```

这样的处理方式，可以保证同一个JVM里面，即使反序列化，也是单例的，但是不同JVM间的枚举单例hashcode是不一致的，但是这也不影响是单例。

# 总结

实际在非枚举单例里面，都没处理好反射和反序列化的问题，枚举也提供了一个样例让其他的单例实现学习解决这个问题。

# TODO

为普通的单例补充反射、反序列化的实现方法



# 代码

 https://github.com/Wooyulin/yulam-demo.git   

# 参考

https://leokongwq.github.io/2017/08/21/why-enum-singleton-are-serialization-safe.html

https://www.jianshu.com/p/5b69dd7ebde1

https://www.cnblogs.com/chiclee/p/9097772.html