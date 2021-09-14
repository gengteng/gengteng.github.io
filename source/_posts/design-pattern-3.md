---
title: 设计模式（三）[WIP]
date: 2021-09-14 14:10:37
tags: ['Design Pattern']
---

## 装饰器（Decorator）

> Component：定义一个对象接口，可以给这些对象动态地添加职责。  
> ConcreteComponent：定义对象，可以给这个对象添加一些职责。  
> Decorator：维持一个指向 Component 对象的指针，并定义一个与 Component 接口一致的接口。  
> ConcreteDecorator：实际的装饰对象，向组件添加职责。  

* Java 实现

```java
public class DecoratorPattern {

    public static void main(String[] args) {
        Component component0 = new ADecorator(new BDecorator(new Component0()));
        component0.doSth();
        System.out.println();
        Component component1 = new BDecorator(new ADecorator(new BDecorator(new ADecorator(new Component1()))));
        component1.doSth();
        System.out.println();
    }

    public interface Component {
        void doSth();
    }

    public static class Component0 implements Component {

        @Override
        public void doSth() {
            System.out.print("0");
        }
    }

    public static class Component1 implements Component {

        @Override
        public void doSth() {
            System.out.print("1");
        }
    }

    public static abstract class Decorator implements Component {
        protected Component component;
        public Decorator(Component component) {
            this.component = component;
        }
    }

    public static class ADecorator extends Decorator {

        public ADecorator(Component component) {
            super(component);
        }

        @Override
        public void doSth() {
            component.doSth();
            System.out.print("A");
        }
    }

    public static class BDecorator extends Decorator {

        public BDecorator(Component component) {
            super(component);
        }

        @Override
        public void doSth() {
            component.doSth();
            System.out.print("B");
        }
    }
}
```

* Rust 实现

```rust
pub trait Component {
        fn do_sth(&self);
    }

    pub struct Component1;

    impl Component for Component1 {
        fn do_sth(&self) {
            print!("1");
        }
    }

    pub struct Component2;

    impl Component for Component2 {
        fn do_sth(&self) {
            print!("2");
        }
    }

    pub trait Decorator<T: Component>: Component {
        fn wrap(inner: T) -> Self;
    }

    struct DecoratorA<T> {
        inner: T,
    }

    impl<T: Component> Component for DecoratorA<T> {
        fn do_sth(&self) {
            self.inner.do_sth();
            print!("A");
        }
    }

    impl<T: Component> Decorator<T> for DecoratorA<T> {
        fn wrap(inner: T) -> Self {
            Self { inner }
        }
    }

    struct DecoratorB<T> {
        inner: T,
    }

    impl<T: Component> Component for DecoratorB<T> {
        fn do_sth(&self) {
            self.inner.do_sth();
            print!("B");
        }
    }

    impl<T: Component> Decorator<T> for DecoratorB<T> {
        fn wrap(inner: T) -> Self {
            Self { inner }
        }
    }

    #[test]
    fn test_decorator() {
        let c1 = Component1;
        c1.do_sth();
        println!();
        let ab1 = DecoratorA::wrap(DecoratorB::wrap(Component1));
        ab1.do_sth();
        println!();
        let abbaa2 = DecoratorA::wrap(DecoratorB::wrap(DecoratorB::wrap(DecoratorA::wrap(
            DecoratorA::wrap(Component2),
        ))));
        abbaa2.do_sth();
        println!();

        // 在Rust中，如果不需要运行时动态装饰，就没有必要产生很多小对象
        // 装饰了好几轮，最后占用内存还是 Component2 的大小
        assert_eq!(
            std::mem::size_of::<
                DecoratorA<DecoratorB<DecoratorB<DecoratorA<DecoratorA<Component2>>>>>,
            >(),
            0
        );
    }
```

输出内容：

```
1
1BA
2AABBA
```

> 优点：比继承更灵活，避免类爆炸；可以组合；装饰器和构件可以独立变化，符合开闭原则。  
> 缺点：产生很多小对象，增加系统复杂度和理解难度，调试困难。

----

