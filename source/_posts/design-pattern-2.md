---
title: 设计模式（二）
date: 2021-08-24 10:55:32
tags: ['Design Pattern']
---

## 策略模式（Strategy）

策略模式是对算法的包装，是把使用算法的责任和算法本身分割开来，委派给不通过的对象管理。

最典型的例子就是使用比较器（`Comparator<T>`）：

```java
@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);

    // 省略其他...
}
```

```java
import java.util.Arrays;
import java.util.Comparator;

public class Cat {
    int weight;

    public int GetWeight() {
        return weight;
    }

    public Cat(int weight) {
        this.weight = weight;
    }

    @Override
    public String toString() {
        return "Cat{" +
                "weight=" + weight +
                '}';
    }

    public static void main(String[] args) {
        Cat[] cats = {new Cat(12), new Cat(21), new Cat(4), new Cat(8)};

        // 这里可以new一个实现了Comparator的类，也可以使用Lambda表达式
        Arrays.sort(cats, Comparator.comparingInt(Cat::GetWeight));

        System.out.println(Arrays.toString(cats));
    }
}
```

> 注：
> 1. 开闭原则（OCP，Open-Closed Principle），对修改关闭，对扩展开放。
> 2. 自定义静态泛型函数时，将类型参数列表写在 `static` 和返回值类型之间，例如：
> ```java
> static <T> void sort(T[] array, Comparator c);
> ```

----

## 抽象工厂（Abstract Factory）

一种为访问类提供一个创建一组相关或相互依赖对象的接口，且访问类无须指定所要产品的具体类就能得到同族的不同等级的产品的模式结构。

例如，农场可以生产动物、蔬菜。农场A生产牛，白菜，农场B生产羊，花椰菜。编码如下：

```java
// 农场抽象
public abstract class Farm {
    public abstract Animal createAnimal();
    public abstract Vegetable createVegetable();
}

// 动物抽象
public interface Animal {
}

// 蔬菜抽象
public interface Vegetable {
}
```

```java
// 农场A
public class FarmA extends Farm {
    @Override
    public Animal createAnimal() {
        return new Cow();
    }

    @Override
    public Vegetable createVegetable() {
        return new Cabbage();
    }
}

public class Cow implements Animal {
}

public class Cabbage implements Vegetable {
}
```

```java
// 农场B
public class FarmB extends Farm {
    @Override
    public Animal createAnimal() {
        return new Sheep();
    }

    @Override
    public Vegetable createVegetable() {
        return new Cauliflower();
    }
}

public class Sheep implements Animal {
}

public class Cauliflower implements Vegetable {
}
```

```java
// 实际调用
public class Main {
    public static void main(String[] args) {
        Farm farm = new FarmA();
        System.out.println(farm.createAnimal().getClass().getSimpleName());
        System.out.println(farm.createVegetable().getClass().getSimpleName());
    }
}
```

输出：

```
Cow
Cabbage
```

使用 Rust 语言编写如下：

```rust
use std::fmt::Debug;

// 产品抽象
trait Animal {}
trait Vegetable {}

// 工厂抽象
trait Farm {
    type A: Animal;
    type V: Vegetable;
    const NAME: &'static str;

    fn create_animal() -> Self::A;
    fn create_vegetable() -> Self::V;
}

#[derive(Debug)]
struct Cow;
impl Animal for Cow {}
#[derive(Debug)]
struct Cabbage;
impl Vegetable for Cabbage {}
struct FarmA;
impl Farm for FarmA {
    type A = Cow;
    type V = Cabbage;
    const NAME: &'static str = "A";

    fn create_animal() -> Self::A {
        Cow
    }

    fn create_vegetable() -> Self::V {
        Cabbage
    }
}

#[derive(Debug)]
struct Sheep;
impl Animal for Sheep {}
#[derive(Debug)]
struct Cauliflower;
impl Vegetable for Cauliflower {}
struct FarmB;
impl Farm for FarmB {
    type A = Sheep;
    type V = Cauliflower;
    const NAME: &'static str = "B";

    fn create_animal() -> Self::A {
        Sheep
    }

    fn create_vegetable() -> Self::V {
        Cauliflower
    }
}

// 实际调用
fn run_a_farm<F>()
where
    F: Farm,
    F::A: Debug,
    F::V: Debug,
{
    println!("Farm {}:", F::NAME);
    println!("\tAnimal: {:?}", F::create_animal());
    println!("\tVegetable: {:?}", F::create_vegetable());
}

fn main() {
    run_a_farm::<FarmA>();
    run_a_farm::<FarmB>();
}
```

输出结果为：

```
Farm A:
	Animal: Cow
	Vegetable: Cabbage
Farm B:
	Animal: Sheep
	Vegetable: Cauliflower
```

### 典型应用场景

1. 界面换肤，一款皮肤下，不同的控件的样式；

> 注：
> 1. 增加新的产品族，只需要新增工厂实现，满足开闭原则；
> 2. 产品族需要新增产品的时候，需要修改所有工厂，不满足开闭原则。
> 3. 当系统只存在一类产品时，抽象工厂退化到工厂方法模式。

----

## 外观/门面（Facade）

通过为多个复杂的子系统提供一个一致的接口，而使这些子系统更加容易被访问的模式。

它是迪米特法则的典型应用。降低了子系统与客户端之间的耦合度。

> 迪米特法则（LOD，Law of Demeter）又叫作最少知识原则（The Least Knowledge Principle），一个类对于其他类知道的越少越好，就是说一个对象应当对其他对象有尽可能少的了解,只和朋友通信，不和陌生人说话。

