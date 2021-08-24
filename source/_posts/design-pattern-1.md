---
title: 设计模式（一）
date: 2021-08-24 10:55:32
tags: ['Design Pattern']
---

## 单例模式

三种最典型的懒加载单例写法。

### 双重检查锁

```java
public class Singleton {

    // 定义为 volatile，防止指令重排序，导致未完成初始化的对象被使用
    private static volatile Singleton INSTANCE = null;

    public static Singleton getInstance() {
        // 基于性能考量，第一次检查，避免每一次都加锁
        if (INSTANCE == null) {
            // 加锁
            synchronized (Singleton1.class) {
                // 检查这之前是否有其他线程已经获得过锁并初始化
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }

        return INSTANCE;
    }
}
```

### 静态内部类

```java
public class Singleton {

    // 外部类被加载时，内部类还不会被加载
    private static class SingletonHolder {
        private final static Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        // 调用这里时，内部类被加载，实现初始化
        // 由JVM保证只加载一次，线程安全
        return SingletonHolder.INSTANCE;
    }

    private Singleton() {}
}
```

### 枚举

```java
// 懒加载，线程安全，还可以防止反序列化
public enum Singleton {
    INSTANCE;

    public static Singleton getInstance() {
        return Singleton.INSTANCE;
    }
}
```