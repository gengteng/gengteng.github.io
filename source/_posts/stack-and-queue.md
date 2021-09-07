---
title: 栈和队列
date: 2021-09-06 15:45:08
tags: ['algorithm', 'queue', 'stack', 'ring buffer']
---


### 1. 环形队列

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

### 2. 时间复杂度为O(1)的栈中最小值获取方法

* Java 实现

```java
import java.util.Collection;
import java.util.Collections;
import java.util.Optional;
import java.util.Stack;

public class GetMinStack<E extends Comparable<E>> extends Stack<E> {
    private final Stack<E> minStack;

    public static void main(String[] args) {
        GetMinStack<Integer> stack = new GetMinStack<>();
        int[] data = {8, 4, 2, 6, 1, 9, -1, 3};
        for (int i: data) {
            stack.push(i);
            Optional<Integer> min = stack.getMin();
            System.out.println("push " + i + ", min: " + min.get() + ", stack" + stack + ", minStack: " + stack.getMinStack());
        }

        while (!stack.empty()) {
            int i = stack.pop();
            Optional<Integer> min = stack.getMin();
            if (min.isPresent()) {
                System.out.println("pop " + i + ", min: " + min.get() + ", stack" + stack + ", minStack: " + stack.getMinStack());
            } else {
                System.out.println("pop " + i + ", stack is empty");
            }
        }
    }

    public GetMinStack() {
        minStack = new Stack<>();
    }

    @Override
    public synchronized E pop() {
        E item = super.pop();
        E min = minStack.peek();
        // 如果出栈的元素跟最小栈顶元素相等，则最小栈顶也出栈
        if (min == item) {
            minStack.pop();
        }
        return item;
    }

    @Override
    public E push(E item) {
        if (!minStack.empty()) {
            E min = minStack.peek();
            // 如果栈不空，看最小栈顶与入栈元素哪个小；
            // 一样大或者入栈元素小，则该元素入最小栈；
            // 否则不做任何操作
            if (min.compareTo(item) >= 0) {
                minStack.push(item);
            }
        } else {
            // 栈空就直接入栈
            minStack.push(item);
        }
        return super.push(item);
    }

    public Optional<E> getMin() {
        if (empty()) {
            return Optional.empty();
        } else {
            return Optional.of(minStack.peek());
        }
    }

    public Collection<E> getMinStack() {
        return Collections.unmodifiableCollection(this.minStack);
    }
}
```

输出内容：

```
push 8, min: 8, stack[8], minStack: [8]
push 4, min: 4, stack[8, 4], minStack: [8, 4]
push 2, min: 2, stack[8, 4, 2], minStack: [8, 4, 2]
push 6, min: 2, stack[8, 4, 2, 6], minStack: [8, 4, 2]
push 1, min: 1, stack[8, 4, 2, 6, 1], minStack: [8, 4, 2, 1]
push 9, min: 1, stack[8, 4, 2, 6, 1, 9], minStack: [8, 4, 2, 1]
push -1, min: -1, stack[8, 4, 2, 6, 1, 9, -1], minStack: [8, 4, 2, 1, -1]
push 3, min: -1, stack[8, 4, 2, 6, 1, 9, -1, 3], minStack: [8, 4, 2, 1, -1]
pop 3, min: -1, stack[8, 4, 2, 6, 1, 9, -1], minStack: [8, 4, 2, 1, -1]
pop -1, min: 1, stack[8, 4, 2, 6, 1, 9], minStack: [8, 4, 2, 1]
pop 9, min: 1, stack[8, 4, 2, 6, 1], minStack: [8, 4, 2, 1]
pop 1, min: 2, stack[8, 4, 2, 6], minStack: [8, 4, 2]
pop 6, min: 2, stack[8, 4, 2], minStack: [8, 4, 2]
pop 2, min: 4, stack[8, 4], minStack: [8, 4]
pop 4, min: 8, stack[8], minStack: [8]
pop 8, stack is empty
```

### 3. 用栈实现队列

* Java 实现

```java
import java.util.Optional;
import java.util.Stack;

public class StackQueue<E> {
    // 用来入队
    private final Stack<E> pushStack;
    // 用来出队
    private final Stack<E> popStack;

    public static void main(String[] args) {
        StackQueue<String> queue = new StackQueue<>();
        String[] data = {"hello", "world", "how", "are", "you"};
        for (String s: data) {
            queue.push(s);
        }
        System.out.println(queue);
        Optional<String> op = queue.poll();
        while (op.isPresent()) {
            System.out.println(op.get());
            op = queue.poll();
        }
        System.out.println(queue);
        for (String s: data) {
            queue.push(s);
        }
        System.out.println(queue);
        op = queue.poll();
        for (String s: data) {
            queue.push(s);
        }
        System.out.println(queue);
        while (op.isPresent()) {
            System.out.println(op.get());
            op = queue.poll();
        }
        System.out.println(queue);
    }

    public StackQueue() {
        pushStack = new Stack<>();
        popStack = new Stack<>();
    }

    public int size() {
        return pushStack.size() + popStack.size();
    }

    public boolean empty() {
        return pushStack.empty() && popStack.empty();
    }

    public boolean contains(Object o) {
        return pushStack.contains(o) || popStack.contains(o);
    }

    public void push(E element) {
        pushStack.push(element);
    }

    public Optional<E> poll() {
        if (popStack.empty()) {
            while (!pushStack.empty()) {
                popStack.push(pushStack.pop());
            }
            if (popStack.empty()) {
                return Optional.empty();
            }
        }

        return Optional.of(popStack.pop());
    }

    @Override
    public String toString() {
        // => 表示栈顶
        return "StackQueue{" +
                "pushStack=" + pushStack +
                "=>, popStack=" + popStack +
                "=>}";
    }
}
```

输出内容：

```
StackQueue{pushStack=[hello, world, how, are, you]=>, popStack=[]=>}
hello
world
how
are
you
StackQueue{pushStack=[]=>, popStack=[]=>}
StackQueue{pushStack=[hello, world, how, are, you]=>, popStack=[]=>}
StackQueue{pushStack=[hello, world, how, are, you]=>, popStack=[you, are, how, world]=>}
hello
world
how
are
you
hello
world
how
are
you
StackQueue{pushStack=[]=>, popStack=[]=>}
```

### 4. 用队列实现栈

* Java 实现

```java
import java.util.LinkedList;
import java.util.Optional;
import java.util.Queue;

public class QueueStack<E> {
    private Queue<E> data;
    private Queue<E> help;

    public static void main(String[] args) {
        QueueStack<String> stack = new QueueStack<>();
        String[] data = {"hello", "world", "how", "are", "you"};
        for (String s: data) {
            stack.push(s);
        }
        Optional<String> op = stack.pop();
        while (op.isPresent()) {
            System.out.println(op.get());
            op = stack.pop();
        }
    }

    public QueueStack() {
        data = new LinkedList<>();
        help = new LinkedList<>();
    }

    public boolean empty() {
        return data.isEmpty() && help.isEmpty();
    }

    public boolean contains(E object) {
        return data.contains(object) || help.contains(object);
    }

    public int size() {
        return data.size() + help.size();
    }

    public void push(E e) {
        data.add(e);
    }

    public Optional<E> pop() {
        int size = data.size();
        if (size == 0) {
            return Optional.empty();
        }

        for (int i=0;i<size-1;++i) {
            help.add(data.poll());
        }

        E e = data.poll();

        // swap
        Queue<E> temp = help;
        help = data;
        data = temp;

        return Optional.of(e);
    }
}
```

输出内容：

```
you
are
how
world
hello
```