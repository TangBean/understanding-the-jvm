# 实现 Java 类的热替换

<!-- TOC -->

- [实现 Java 类的热替换](#实现-java-类的热替换)
    - [什么是热替换及其实现原理](#什么是热替换及其实现原理)
    - [实现简单的 Java 类热替换](#实现简单的-java-类热替换)
        - [需求分析](#需求分析)
        - [实现类加载器](#实现类加载器)

<!-- /TOC -->

## 什么是热替换及其实现原理

- 热替换是在不停止正在运行的系统的情况下进行类（对象）的升级替换；
- 这要求虚拟机中要存在同一个类的两个不同版本。可我们知道，我们是无法将同一个类加载两遍的，想要实现这点，我们需要让虚拟机认为这是两个不同的类，即用两个不同的类加载器去加载这个类不同版本的 class 文件；
- 因此，这个工作就不能由系统提供给我们的启动类加载器，扩展类加载器或者应用程序类加载器来完成，因为这三个类加载器在同一个虚拟机中只有一份，不仅如此，我们还要跳过这些类加载器；
- 想要跳过这些类加载器可不是只要不用这些类加载器就行了，还需要我们跳过双亲委派模型，否则类的加载还会被委派到这些个类加载器，如果恰好某个类之前是由这三个类加载器中的一个加载的，虚拟机就不会再次加载新版本的类了，就无法实现类的热替换了。



## 实现简单的 Java 类热替换

### 需求分析

现有一 Foo 类，可以在控制台持续打印：Hello world! version one，我们将在该类运行时，将其 .class 文件替换为修改后的 Foo 类的 .class 文件，修改后的 Foo 会在控制台持续打印：Hello world! version two。也就是说，替换之后，控制台打印的内容发生变化，就说明类的热替换实现成功。

Foo 类的实现：

```java
public class Foo {
    public void sayHello() {
        System.out.println("Hello world! version one");
        // System.out.println("Hello world! version two");  // 之后替换成这个
    }
}
```

然后我们通过如下程序运行 Foo 类：

```java
public class Task extends TimerTask {
    @Override
    public void run() {
        String basePath = "C:\\Users\\Bean\\IdeaProjects\\USTJ\\target\\classes";
        // 每执行一次任务都 new 一个新的类加载器
        HotswapClassLoader cl = new HotswapClassLoader(
            basePath, new String[]{"com.jvm.ch7.hotswap.Foo"});
        try {
            // 通过我们自己实现的类加载器加载 Foo 类
            Class cls = cl.loadClass("com.jvm.ch7.hotswap.Foo", true); 
            Object foo = cls.newInstance();
            Method method = cls.getMethod("sayHello", new Class[]{});
            method.invoke(foo, new Object[]{});
        } catch (Exception e) {
        	e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        Timer timer = new Timer();
        timer.schedule(new Task(), 0, 1000);
    }
}
```

### 实现类加载器

`HotswapClassLoader` 的实现如下，具体的讲解已被写入注释中：

```java
public class HotswapClassLoader extends ClassLoader {
    private String basePath;
    private HashSet<String> loadedClass;  // 用来记录被这个类加载器所加载的类

    public HotswapClassLoader(String basePath, String[] classList) {
        // 跳过父类加载器，把它设为null
        super(null);
        this.basePath = basePath;
        loadedClass = new HashSet<>();
        // 该类加载器在初始化的时候会直接把应该它负责加载的类加载好，
        // 这样之后 loadClass 时，会在第一步检验该类是否已经被加载时发现该类已经被加载过了，
        // 就无需执行 loadClass 之后的流程，直接返回虚拟机中被加载好的类即可，
        // 这样虽然初始化的时间长了点，但是之后 loadClass 时会比较省时间
        loadClassByMe(classList);
    }

    /**
     * 加载给定的的 classList 中的类到虚拟机
     */
    private void loadClassByMe(String[] classList) {
        for (int i = 0; i < classList.length; i++) {
            Class cls = loadClassDirectly(classList[i]);
            if (cls != null) {
                loadedClass.add(classList[i]);
            }
        }
    }

    /**
     * 通过文件名直接加载类，得到Class对象
     */
    private Class loadClassDirectly(String className) {
        Class cls = null;
        StringBuilder sb = new StringBuilder(basePath);
        String classPath = className.replace(".", File.separator) + ".class";
        sb.append(File.separator + classPath);
        File file = new File(sb.toString());
        InputStream fin = null;
        try {
            fin = new FileInputStream(file);
            // 将字节流转化成内存中的Class对象
            cls = instantiateClass(className, fin, (int) file.length());
            return cls;
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (fin != null) {
                try {
                    fin.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return null;
    }

    /**
     * 将字节流转化成内存中的Class对象啊，使用defineClass方法！
     */
    private Class instantiateClass(String name, InputStream fin, int len) {
        byte[] buffer = new byte[len];
        try {
            fin.read(buffer);
            return defineClass(name, buffer, 0, len);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (fin != null) {
                try {
                    fin.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return null;
    }

    /**
     * 覆盖原有的loadClass规则，
     */
    public Class loadClass(String name, boolean resolve) throws ClassNotFoundException {
        Class cls = null;
        // 应该由 HotswapClassLoader 负责加载的类会通过下面这一行得到类的 Class 对象，
        // 因为早在 HotswapClassLoader 类加载器执行构造函数时，它们就被加载好了
        cls = findLoadedClass(name);
        // 只有在这个类没有被加载，且！这个类不是当前这个类加载器负责加载的时候，才去使用启动类加载器
        if (cls == null && !loadedClass.contains(name)) {
            cls = findSystemClass(name);
        }
        if (cls == null) {
            throw new ClassNotFoundException(name);
        }
        // resolveClass是进行连接操作的，即"验证+准备+解析"，之后就可以进行初始化了
        if (resolve) {
            resolveClass(cls);
        }
        return cls;
    }
}
```

