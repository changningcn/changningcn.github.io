---
layout: post
title: Java ClassLoader
tag: Java
---

## 前言
由于一直对 ```ClassLoader``` 及反射没有加深了解，刚好听到同事有个需求，希望能根据配置文件得到因转换为 Json，又由 Json 转化为 Object 获取实际类的对象。

先写一个 ClassLoader

```java

import com.google.gson.internal.LinkedTreeMap;
import java.lang.reflect.Constructor;
import java.lang.reflect.Method;

public class JavaClassLoader extends ClassLoader {

    public Object invokeClassMethod(String classBinName, String methodName, LinkedTreeMap o){

        try {

            // Create a new JavaClassLoader
            ClassLoader classLoader = this.getClass().getClassLoader();

            // Load the target class using its binary name
            Class loadedMyClass = classLoader.loadClass(classBinName);

            System.out.println("Loaded class name: " + loadedMyClass.getName());

            // Create a new instance from the loaded class
            Constructor constructor = loadedMyClass.getConstructor(LinkedTreeMap.class);
            Object  myClassObject = constructor.newInstance(o);

            // Getting the target method from the loaded class and invoke it using its name
            Method method = loadedMyClass.getMethod(methodName);
            System.out.println("Invoked method name: " + method.getName());
            return method.invoke(myClassObject);

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;

    }
}

```
然后是个测试类, ```JavaCodeTest``` 中有个 int a 以及 getter、setter，其中注意的是还有个参数为 ```LinkedTreeMap``` 的构造函数，这是因为从```gson.fromJson(s, Object.class);```中返回的就是```LinkedTreeMap```，而且无法直接由 ```Object``` 转化```JavaCodeTest```，所以在调用```getConstructor```时，要把对应属性获得，就应该在构造函数中进行赋值(```convert()```)，这样生成的实例(```newInstance(o)```)时就可以得到想要的类了。

```java
import com.google.gson.Gson;
import com.google.gson.internal.LinkedTreeMap;
import java.lang.annotation.Annotation;
import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class JavaCodeTest {

    public JavaCodeTest(Object o){
        JavaCodeTest javaCodeTest = convert(o);
        setA(javaCodeTest.getA());
    }

    public JavaCodeTest(LinkedTreeMap l){
        setA((Double) l.get("a"));
    }

    public JavaCodeTest(){
    }


    JavaCodeTest convert(Object o){
        return (JavaCodeTest) o;
    }

    private double a = 0;
    private List list = new ArrayList<JavaCodeTest>();
    private Map map = new HashMap<String, JavaCodeTest>();

    public void setA(double a) {
        this.a = a;
    }

    public double getA() {
        return a;
    }

    public static void main(String[] args){

        JavaCodeTest javaCodeTest = new JavaCodeTest();
        JavaCodeTest javaCodeTest1 = new JavaCodeTest();
        javaCodeTest1.map.put("test", new JavaCodeTest());
        javaCodeTest.setA(100);
        javaCodeTest.list.add(javaCodeTest1);
        Gson gson = new Gson();
        String s = gson.toJson(javaCodeTest);
        LinkedTreeMap o = (LinkedTreeMap) gson.fromJson(s, Object.class);

        JavaClassLoader javaClassLoader = new JavaClassLoader();
        javaClassLoader.invokeClassMethod("JavaCodeTest", "getA", o);
    }

}

```