# 聊聊Java反射

## 1 Class类
### 1.1 万事万物皆对象 
在面向对象的世界里，万事万物皆对象

注意：Java语言中，

        普通数据类型类，如int a = 5，不是对象，但是它有包装类或者封装类来弥补它

        静态的成员，不是面向对象的，它是属于某个类的

*Question1*：类是谁的对象呢？

*Answer1*：  类是对象，类是java.lang.Class类的实例对象

*Question2*：这个对象如何表示呢？

*Answer2*：  
```java
package com.sjf.reflect;

public class Test {
    public static void main(String[] args) {
        // Foo的实例对象如何表示？
        Foo foo1 = new Foo(); // foo1就表示出来了

        // Foo这个类也是一个实例对象，Class类的实例对象，如何表示呢？
        // 任何一个类都是Class的实例对象，这个实例对象有三种表示方式

        // 第一种表述方式
        //实际在告诉我们任何一个类都有一个隐含的静态成员变量class
        Class c1 = Foo.class;

        // 第二种表达方式
        //已经知道该类的对象通过getClass方法
        Class c2 = foo1.getClass();

        /* 官网说明 c1,c2表示了Foo类的类类型(class type)
         * 万事万物皆对象
         * 类也是对象，是Class类的实例对象
         * 这个对象我们称为该类的类类型
         */

        // 一个类只可能是Class类的一个实例对象
        System.out.println(c1 == c2);

        // 第三种表达方式
        Class c3 = null;
        try {
            c3 = Class.forName("com.sjf.reflect.Foo");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

        // 可以通过类的类类型创建该类的对象实例
        // -->通过c1,c2,c3创建Foo的实例对象
        try {
            // 前提是需要有无参数的构造方法
            Foo foo = (Foo) c1.newInstance();  
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}

class Foo {

}
```

### 1.2 Class动态加载类
Class.forName("类的全称")
- 不仅表示了类的类类型，还代表了动态加载类
- 请大家区分编译、运行
- 编译时刻加载类是静态加载类、运行时刻加载类是动态加载类

```java
class Office {
    public static void main(String[] args) {
        if ("Word".equals(args[0])) {
            Word w = new Word();
            w.start();
        }
        if ("Excel".equals(args[0])) {
            Excel e = new Excel();
            e.start();
        }
    }
}
```
这么写代码是有问题的，由于没有Word类和Excel类，程序无法通过编译。
但是，Word类一定用吗？Excel类一定用吗？或者我现在提供了Word类
```java
class Word {
    public void start() {
        System.out.println("Word starts");
    }
}
```
程序依然会因为缺少Excel类而无法通过编译，这样的程序显然是缺少灵活性的。
问题的根源在于：
*new创建对象*是静态加载类，在编译时刻就需要加载所有可能使用到的类。
通过动态加载类可以解决上述问题：
```java 
class BetterOffice {
    public static void main(String[] args) {
        try {
            // 动态加载类，在运行时刻加载
            Class c = Class.forName(args[0]);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
解决了上述问题，下面通过类类型，创建该类的对象：
```java 
class BetterOffice {
    public static void main(String[] args) {
        try {
            // 动态加载类，在运行时刻加载
            Class c = Class.forName(args[0]);
            // 通过类类型，创建该类对象
            Word w = (Word) c.newInstance();
            w.start();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
观察上面的代码，发现如果以后要添加更多的对象，其实是比较麻烦的。
这个时候应该想到多态可以解决问题：
```java
class BetterOffice {
    public static void main(String[] args) {
        try {
            // 动态加载类，在运行时刻加载
            Class c = Class.forName(args[0]);
            // 通过类类型，创建该类对象
            OfficeAble oa = (OfficeAble) c.newInstance();
            oa.start();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

public Interface OfficeAble {
    void start();
}
class Word implements OfficeAble {
    public void start() {
        System.out.println("Word starts");
    }
}
```

# 2 成员函数、成员变量、构造函数的反射
## 2.1 获取方法信息 
成员函数也是对象

java.lang.reflect.Method封装了关于成员函数的操作

getMethods()方法获取的是所有的public的函数，包括父类继承而来的

getDeclaredMethods()获取的是所有该类自己声明的方法，不论访问权限

```java
public class Test {
    public static void main(String[] args) {
        Class c1 = int.class;
        Class c2 = String.class;
        Class c3 = double.class;
        Class c4 = Double.class;
        Class c5 = void.class;

        System.out.println(c1.getName());
        System.out.println(c2.getName());
        System.out.println(c2.getSimpleName());
        System.out.println(c3.getName());
        System.out.println(c4.getName());
        System.out.println(c5.getName());
    }
}
```

```java
import java.lang.reflect.Method;

public class ClassUtil {
    /*
     * 打印类的信息：类的成员函数
     * obj 该对象所属类的信息
     */
    public static void printClassMessage(Object obj) {
        // 要获取类的信息，首先要获取类的类类型
        Class c = obj.getClass();  
        // 传递的是哪个子类的对象 c就是该子类的类类型

        // 获取类的名称
        System.out.println("类的名称是：" + c.getName());

        /*
         * Method类，方法对象
         * 一个成员方法就是一个Method对象
         * getMethods()方法获取的是所有的public的函数，包括父类继承而来的
         * getDeclaredMethods()获取的是所有该类自己声明的方法，不论访问权限
         */
        Method[] ms = c.getMethods(); // c.getDeclaredMethods();

        for (int i = 0; i < ms.length; i++) {
            // 得到方法的返回值类型的类类型
            Class returnType = ms[i].getReturnType(); 

            System.out.print(returnType.getName() + " ");

            // 得到方法的名称
            System.out.print(ms[i].getName() + "(");

            // 获取参数类型-->得到的是参数列表的类型的类类型
            Class[] paramTypes = ms[i].getParameterTypes();
            for (Class class1 : paramTypes) {
                System.out.print(class1.getName() + ",");
            }
            System.out.println(")");
        }
    }
}

public class ClassDemo {
    public static void main(String[] args) {
        String s = "hello";
        ClassUtil.printClassMessage(s);
    }
}
```

## 2.2 获取成员变量
成员变量也是对象

java.lang.reflect.Field封装了关于成员变量的操作

getFields()方法获取的是所有的public的成员变量的信息

getDeclaredFields获取的是该类自己声明的成员变量的信息

```java 
public class ClassUtil {
    /*
     * 打印类的信息：成员变量
     * obj 该对象所属类的信息
     */
    public static void printFieldMessage(Object obj) {
        // 要获取类的信息，首先要获取类的类类型
        Class c = obj.getClass();  // 传递的是哪个子类的对象 c就是该子类的类类型

        Field[] fs = c.getDeclaredFields();
        for (Field field : fs) {
            // 得到成员变量的类型的类类型
            Class fieldType = field.getType();
            String typeName = fieldType.getName();
            // 得到成员变量的名称
            String fieldName = field.getName();
            System.out.println(typeName + " " + fieldName);
        }
    }
}
```

## 2.3 获取构造函数
构造函数也是对象

java.lang.reflect.Constructor中封装了构造函数的信息

getConstructors获取所有的public的构造函数

getDeclaredConstructors获取所有的构造函数

```java 
public class ClassUtil {
    /*
     * 打印类的信息：构造函数
     * obj 该对象所属类的信息
     */
    public static void printConMessage(Object obj) {
        // 要获取类的信息，首先要获取类的类类型
        Class c = obj.getClass();  // 传递的是哪个子类的对象 c就是该子类的类类型

        // Constructor[] cs = c.getConstructors();
        Constructor[] cs = c.getDeclaredConstructors();

        for (Constructor constructor : cs) {
            System.out.println(costructor.getName() + "(");
            // 获取构造函数的参数列表-->得到的是参数列表的类型的类类型
            Class[] paramTypes = constructor.getParameterTypes();
            for (Class class1 : paramTypes) {
                System.out.print(class1.getName() + ",");
            }
            System.out.println(")");
        }
    }
}

public class ClassDemo {
    public static void main(String[] args) {
        String s = "hello";
        ClassUtil.printConMessage(s);
    }
}
```

# 3 方法的反射 
1) 如何获取某个方法
   方法的名称和方法的参数列表才能唯一决定某个方法
2) 方法反射的操作
   method.invoke(对象,参数列表)

```java 
import java.lang.reflect.Method;
public class MethodDemo {
    public static void main(String[] args) {
        // 要获取print(int,int)方法
        // 1.要获取一个方法就是获取类的信息，获取类的信息首先要获取类的类类型
        A a1 = new A();
        Class c = a1.getClass();
        // 2.获取方法 名称和参数列表来决定
        // getMethod获取的是public的方法
        // getDelcaredMethod获取自己声明的方法
        try {
            //c.getMethod("print", new Class[]{int.class,int.class})
            Method m = c.getMethod("print", int.class, int.class);
            
            // 方法的反射操作
            // 原来的做法：a1.print(10, 20);
            // 现在的做法：用m对象来进行方法调用 
            // 方法如果没有返回值返回null，有返回值返回具体的返回值
            //Object o = m.invoke(a1, new Object[]{10, 20});
            Object o = m.invoke(a1, 10, 20);

            System.out.println("========================");

            Method m1 = c.getMethod("print", String.class, String.class);
            Object o1 = m1.invoke(a1, "ds", "grg");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

class A {
    public void print(int a, int b) {
        System.out.println(a + b);
    }

    public void print(String a, String b) {
        System.out.println(a.toUpperCase() + "," + b.toLowerCase());
    }
}
```


# 4 通过反射了解集合泛型的本质
通过Class、Method来认识泛型的本质：
```java 
import java.util.ArrayList;
import java.lang.reflect.Method;
public class Test {
    public static void main(String[] args) {
        ArrayList list = new ArrayList();

        ArrayList<String> list1 = new ArrayList<String>();
        list1.add("hello");
        // list1.add(20); 会报错
        Class c1 = list.getClass();
        Class c2 = list1.getClass();
        System.out.println(c1 == c2);

        // 反射操作都是编译之后的操作
        // c1==c2结果返回true说明编译之后集合的泛型是去泛型化的
        // Java中集合的泛型，是防止错误输入的，只在编译阶段有效，
        // 绕过编译就无效了
        // 验证：我们可以通过方法的反射来操作，绕过编译
        try {
            Method m = c1.getMethod("add", Object.class);
            m.invoke(list1, 20);  // 绕过了编译操作就绕过了泛型
            System.out.println(list1.size());
            System.out.println(list1);

            // 现在不能这样遍历了，会有类型转换错误
            /*for (String string : list1) {
                System.out.println(String);
            }*/ 
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

最后，分享一篇关于[Java反射](http://www.infoq.com/cn/articles/cf-java-reflection-dynamic-proxy)的好文。