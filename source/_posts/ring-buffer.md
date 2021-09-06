---
title: 环形队列
date: 2021-09-06 15:45:08
tags: ['algorithm', 'queue', 'ring']
---

* Java 实现

```java
public class RingBuffer<T> {
    // 队列缓冲区
    private final Object[] array;

    // 队列尺寸
    private final int limit;

    // 表示即将入队的索引位置
    private int pushIndex;

    // 表示即将出队的索引位置
    private int popIndex;

    // 表示当前队列中元素的个数
    private int size;

    private RingBuffer(int limit) {
        if (limit <= 0) {
            throw new IllegalArgumentException("limit should be greater than 0");
        }
        this.limit = limit;
        this.array = new Object[limit];
        this.popIndex = this.pushIndex = this.size = 0;
    }

    public static <T> RingBuffer<T> create(int limit) {
        return new RingBuffer<>(limit);
    }

    public Optional<T> pop() {
        if (size == 0) {
            return Optional.empty();
        }

        T value = (T) this.array[this.popIndex];
        this.array[this.popIndex] = null; // 去掉引用，避免泄漏
        this.size -= 1;
        this.popIndex = getNextIndex(this.popIndex);
        return Optional.of(value);
    }

    public boolean empty() {
        return this.size == 0;
    }

    public void push(T value) {
        if (size == this.limit) {
            throw new IllegalArgumentException("The size has reached the limit");
        }

        this.array[this.pushIndex] = value;
        this.size += 1;
        this.pushIndex = getNextIndex(this.pushIndex);
    }

    @Override
    public String toString() {
        return "RingBuffer{" +
                "array=" + Arrays.toString(array) +
                ", limit=" + limit +
                ", pushIndex=" + pushIndex +
                ", popIndex=" + popIndex +
                ", size=" + size +
                '}';
    }

    private int getNextIndex(int index) {
        return index == this.limit - 1 ? 0 : index + 1;
    }

    public static void main(String[] args) {
        RingBuffer<String> rb = RingBuffer.create(4);
        System.out.println("new rb: " + rb);
        String[] data = {"hello", "world", "are", "you", "ok"};
        for (String s: data) {
            try {
                rb.push(s);
                System.out.println("push " + s);
                System.out.println("rb: " + rb);
            } catch (Exception e) {
                System.out.println("Push '" + s + "' error: " + e);
            }
        }

        Optional<String> op = rb.pop();

        while (op.isPresent()) {
            System.out.println("pop " + op.get());
            System.out.println("rb: " + rb);
            op = rb.pop();
        }
    }
}
```

输出内容：

```
new rb: RingBuffer{array=[null, null, null, null], limit=4, pushIndex=0, popIndex=0, size=0}
push hello
rb: RingBuffer{array=[hello, null, null, null], limit=4, pushIndex=1, popIndex=0, size=1}
push world
rb: RingBuffer{array=[hello, world, null, null], limit=4, pushIndex=2, popIndex=0, size=2}
push are
rb: RingBuffer{array=[hello, world, are, null], limit=4, pushIndex=3, popIndex=0, size=3}
push you
rb: RingBuffer{array=[hello, world, are, you], limit=4, pushIndex=0, popIndex=0, size=4}
Push 'ok' error: java.lang.IllegalArgumentException: The size has reached the limit
pop hello
rb: RingBuffer{array=[null, world, are, you], limit=4, pushIndex=0, popIndex=1, size=3}
pop world
rb: RingBuffer{array=[null, null, are, you], limit=4, pushIndex=0, popIndex=2, size=2}
pop are
rb: RingBuffer{array=[null, null, null, you], limit=4, pushIndex=0, popIndex=3, size=1}
pop you
rb: RingBuffer{array=[null, null, null, null], limit=4, pushIndex=0, popIndex=0, size=0}
```