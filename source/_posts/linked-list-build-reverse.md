---
title: 链表的构造和反转
date: 2021-09-06 15:33:40
tags: ['algorithm', 'linked list']
---

## 单向链表

```java
public class Node<T> {
    private final T value;
    private Node<T> next;

    public static void main(String[] args) {
        Node<String> list = Node.build("hello", "world", "are", "you", "ok");
        System.out.println("build: " + list);
        Node<String> reversed = list.reverse();
        System.out.println("reversed: " + reversed);
        Node<String> origin = reversed.reverse();
        System.out.println("origin: " + origin);
    }

    public static <T> Node<T> build(T ...values) {
        Node<T> list = null;
        Node<T> cur = null;

        for (T value: values) {
            if (cur == null) {
                cur = new Node<>(value);
                list = cur;
            } else {
                cur.setNext(new Node<>(value));
                cur = cur.getNext();
            }
        }

        return list;
    }

    public Node(T value) {
        this.value = value;
        this.next = null;
    }

    public Node setNext(Node<T> next) {
        this.next = next;
        return this;
    }

    public Node getNext() {
        return this.next;
    }

    public T getValue() {
        return this.value;
    }

    public Node<T> reverse() {
        Node<T> head = this;
        Node<T> pre = null;

        while (head != null) {
            Node<T> next = head.getNext();

            head.setNext(pre);
            pre = head;

            head = next;
        }

        return pre;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        Node<T> cur = this;
        while (cur != null) {
            sb.append(cur.getValue());
            sb.append(" -> ");
            cur = cur.getNext();
        }
        sb.append("null");
        return sb.toString();
    }
}
```

输出内容：

```
build: hello -> world -> are -> you -> ok -> null
reversed: ok -> you -> are -> world -> hello -> null
origin: hello -> world -> are -> you -> ok -> null
```


## 双向链表

```java
public class DoubleNode<T> {
    private final T value;
    private DoubleNode<T> previous;
    private DoubleNode<T> next;

    public static void main(String[] args) {
        DoubleNode<String> list = DoubleNode.build("hello", "world", "are", "you", "ok");
        System.out.println("build: " + list);
        DoubleNode<String> reversed = list.reverse();
        System.out.println("reversed: " + reversed);
        DoubleNode<String> origin = reversed.reverse();
        System.out.println("origin: " + origin);
        DoubleNode<String> tail = origin.getTailNode();
        System.out.println("back: " + tail.toStringBack());
    }

    public static <T> DoubleNode<T> build(T ...values) {
        DoubleNode<T> list = null;
        DoubleNode<T> cur = null;

        for (T value: values) {
            if (cur == null) {
                cur = new DoubleNode<>(value);
                list = cur;
            } else {
                DoubleNode<T> node = new DoubleNode<>(value);
                node.setPrevious(cur);
                cur.setNext(node);
                cur = cur.getNext();
            }
        }

        return list;
    }

    public DoubleNode(T value) {
        this.value = value;
        this.previous = this.next = null;
    }

    public DoubleNode<T> setNext(DoubleNode<T> next) {
        this.next = next;
        return this;
    }

    public DoubleNode<T> getNext() {
        return this.next;
    }

    public T getValue() {
        return this.value;
    }

    public DoubleNode<T> getTailNode() {
        DoubleNode<T> cur = this;
        while (cur.getNext() != null) {
            cur = cur.getNext();
        }
        return cur;
    }

    public DoubleNode<T> getPrevious() {
        return this.previous;
    }

    public DoubleNode<T> setPrevious(DoubleNode<T> previous) {
        this.previous = previous;
        return this;
    }

    public DoubleNode<T> reverse() {
        DoubleNode<T> head = this;
        DoubleNode<T> pre = null;

        while (head != null) {
            DoubleNode<T> next = head.getNext();

            // 其他都跟单链表一样，指针多设置一个
            head.setNext(pre);
            head.setPrevious(next);
            pre = head;

            head = next;
        }

        return pre;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        DoubleNode<T> cur = this;
        while (cur != null) {
            sb.append(cur.getValue());
            sb.append(" -> ");
            cur = cur.getNext();
        }
        sb.append("null");
        return sb.toString();
    }

    public String toStringBack() {
        StringBuilder sb = new StringBuilder("null");
        DoubleNode<T> cur = this;
        while (cur != null) {
            sb.append(" <- ");
            sb.append(cur.getValue());
            cur = cur.getPrevious();
        }
        return sb.toString();
    }
}
```

输出内容：

```
build: hello -> world -> are -> you -> ok -> null
reversed: ok -> you -> are -> world -> hello -> null
origin: hello -> world -> are -> you -> ok -> null
back: null <- ok <- you <- are <- world <- hello
```