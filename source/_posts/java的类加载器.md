---
title: java的类加载器
date: 2018-03-01 09:38:13
tags: java
subtitle: java
categories: java
---

> ### 类加载器的概念

类加载器顾名思义，就是在程序运行时加载java的字节码文件的一个东西。当我们运行java程序时，编写好的源java文件(以.java为后缀的文件)被编译为java的字节码文件(以.class为后缀的文件)，当运行时需要这个类的时候，类加载器就会把需要的java字节码文件加载并且`newInstance()`生成它的对象，放到jvm虚拟机内存中，让jvm使用。

> ### 类加载器的分类

类加载器这个概念自JDK1.0时就有了，好像一开始sun公司来定义它是因为它可以来加载从网络中取到的java程序。类加载器符合树形结构，如下图：

![](http://ousaim1qx.bkt.clouddn.com/5YTXZK4DT65B%5D$IP%7DS$6%25C9.png)

首先在这里再补充一个概念，类加载器也是类，也是用java语言编写的。好了，那就继续看一下每个类加载器的解释，BootStrap是嵌在JVM内核中的加载器，该加载器是用C++语言写的，主要负载加载JAVA_HOME/lib下的类库，启动类加载器无法被应用程序直接使用。ExtClassLoader又称之为扩展类加载器，它主要来负责加载jre/lib/ext下的类库，如果在此文件夹下有jvm所需要的类，那么此类会被ExtClassLoader来加载。AppClassLoader也是系统类加载器，也是应用程序的类加载器，他负责加载classpath目录下所有的class和jar文件。

> ### 类加载器的工作模式

类加载器并不是通常意义上的继承关系，而是一种双亲委派机制，我通过一张图，来讲解一下，此图为盗用姜维大神的图：

![](http://ousaim1qx.bkt.clouddn.com/20140101125755203.png)

从这张图中，可以很清楚，类加载器的工作机制，在类加载时，自定义的类加载器首先会委托给AppClassLoader，AppClassLoader再委托给ExtClassLoader，最后ExtClassLoader委托BootStrap，BootStrap先去加载，如果加载成功，类加载就结束了，不成功就会交给ExtClassLoader去加载，成功就结束，不成功就继续交给AppClassLoader去加载，成功就结束，不成功继续交给自定义的类加载器，成功就结束，不成功就会报ClassNotFoundException异常，整个类加载过程结束，这也就很好的说明了为什么类加载器的工作原理是双亲委托机制。

> ### java源码中的ClassLoader运行机制

当我们明白java类加载器的工作原理后，我们就可以自定义ClassLoader来满足我们自己的业务需求，虽然，java提供的这三个类加载器已经能满足大多数的业务需求，但是，我们如果想提高程序的安全性那就不得不自定义classloader了，首先自定义classloader都必须要继承ClassLoader这个类。这个类有三个关键的方法，他们是loadClass(),findClass()以及defineClass()。首先我们来看看loadClass()里都做了什么。

```java
 public Class<?> loadClass(String name) throws ClassNotFoundException {
     //返回loadClass(String name, boolean resolve)
        return loadClass(name, false);
    }
```

当类加载要加载类时，他首先会执行loadClass(String name)；这个方法需要一个参数，就是被加载类的名字，这个名字是全信息名。我们继续追踪loadClass(String name, boolean resolve)方法。

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
    //线程锁，防止多线程中同时加载类
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查这个类是否已经被加载了，如果是就不重复加载，否则就加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                /**
                这个if判断非常明显的可以看出为什么说类加载器是双亲委托机制。他会检查这个类加载器的父
                类加载器是否为空如果不为空就调用父类加载器的loadclass方法，否则就执行
                findBootstrapClassOrNull方法，这个方法请看下面。
                */
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    //类没找到，抛出ClassNotFoundException
                }

                if (c == null) {
                    //如果父类的加载器，加载失败就会走到这里
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

```java
private Class<?> findBootstrapClassOrNull(String name)
    {
    //这里就是检查被加载的类名是否正确，不正确返回null
        if (!checkName(name)) return null;
    //这是个native方法看不到源码，可能是用BootStrap来进行类的加载
        return findBootstrapClass(name);
    }
```

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    //哈哈，这就是自定义ClassLoader需要复写的方法
    }
```

```java
/**
这里只需要看最后的defineClass方法就可以了。从中可以看出，加载器是将类变为字节流的形式，读进来，继而进
行类的加载的
*/
protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                         ProtectionDomain protectionDomain)
        throws ClassFormatError
    {
        protectionDomain = preDefineClass(name, protectionDomain);
        String source = defineClassSourceLocation(protectionDomain);
        Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);
        postDefineClass(c, protectionDomain);
        return c;
    }
```

现在清楚了一个ClassLoader中，三个关键方法的执行顺序了首先loadClass()->findClass()->defineClass();

> ### 自己动手定义ClassLoader

首先咱随便写一个需要被我们自定义的类加载器加载的类；

```java
//ClassLoaderJava.java  这是需要被类加载器加载的类，输出一句话；
public class ClassLoaderJava  {
	private static final long VersionId=1L;
	@Override
	public String toString() {
		// TODO Auto-generated method stub
		return "输出类加载器";
	}
}
```

然后我们再自定义ClassLoader；

```java
//    CustomClassLoader.java
public class CustomClassLoader extends ClassLoader{
	private String classPath;

	public CustomClassLoader() {
		// TODO Auto-generated constructor stub
	}

	public CustomClassLoader(String classPath) {
		this.classPath=classPath;
	}
	

	@SuppressWarnings("deprecation")
	@Override
	protected Class<?> findClass(String name) throws ClassNotFoundException {
	// TODO Auto-generated method stub
		int bytes=-1;
		String classPathFile=classPath+"/"+name+".class";
		try {
			FileInputStream fileInputStream=new FileInputStream(classPathFile);
			ByteArrayOutputStream byteArrayOutputStream=new ByteArrayOutputStream();
			while ((bytes=fileInputStream.read())!=-1) {
				byteArrayOutputStream.write(bytes);
			}
			byteArrayOutputStream.flush();
			byteArrayOutputStream.close();
			fileInputStream.close();
			byte[] classByte=byteArrayOutputStream.toByteArray();
			
			return defineClass(classByte, 0, classByte.length);
		} catch (Exception e) {
			// TODO: handle exception
			e.printStackTrace();
		}
		return super.findClass(name);
    }
}
```

最后我们需要一个测试类；

```java
public class Test {
	public static void main(String[] args) {
		// TODO Auto-generated method stub
        try {  
            Class class1 = 
            new CustomClassLoader("bin\\org\\hahah\\demo").loadClass("ClassLoaderJava");  
            Object object = class1.newInstance();
            System.out.println("这是什么ClassLoader:"+object.getClass().getClassLoader().getClass().getName());  
            System.out.println(object);  
        } catch (Exception e1) {  
            e1.printStackTrace();  
        }  
	}

}
```

然后我们运行，Console所打印的信息，正是我们希望看到的结果；

![](http://ousaim1qx.bkt.clouddn.com/9X3H8HQGW%7D%60%5DMUY57~@XD@Y.png)

end！