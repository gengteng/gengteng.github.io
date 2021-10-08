---
title: 堆排序
date: 2021-10-08 15:38:13
tags: ['sort', 'algorithm', 'heap']
---

使用数组结构表示堆结构，元素从索引 0 开始，则索引 `i` 位置：

* 父节点索引为 `(i - 1) / 2`
* 左孩子索引为 `2 * i + 1`
* 右孩子索引为 `2 * i + 2`

----

## 实现

* Java 实现

```java
public class Heap {

    public static void main(String[] args) {
        Heap heap = new Heap(1000);
        heap.push(-6);
        heap.push(3);
        heap.push(-1);
        heap.push(0);
        heap.push(9);
        heap.push(-4);
        heap.push(5);
        heap.push(7);
        heap.push(-2);
        heap.push(9);
        heap.push(13);

        System.out.println(heap);

        while (!heap.isEmpty()) {
            System.out.println(heap.pop());
            System.out.println(heap);
        }
    }

    private final int[] data;
    private final int limit;
    private int heapSize;

    public Heap(int limit) {
        this.limit = limit;
        this.data = new int[limit];
        this.heapSize = 0;
    }

    public void push(int value) {
        if (heapSize >= limit) {
            throw new RuntimeException("heapSize must not exceed limit");
        }

        this.data[heapSize++] = value;
        heapInsert(heapSize - 1);
    }

    public boolean isEmpty() {
        return heapSize == 0;
    }

    public Integer pop() {
        if (isEmpty()) {
            return null;
        }

        swap(--heapSize, 0);
        int result = this.data[heapSize];
        this.data[heapSize] = 0;
        heapify(0);
        return result;
    }

    private void heapInsert(int index) {
        while (index > 0 && this.data[index] > this.data[(index - 1) / 2]) {
            swap(index, (index - 1) / 2);
            index = (index - 1) / 2;
        }
    }

    private void swap(int i, int j) {
        if (i != j) {
            this.data[i] = this.data[i] ^ this.data[j];
            this.data[j] = this.data[i] ^ this.data[j];
            this.data[i] = this.data[i] ^ this.data[j];
        }
    }

    private void heapify(int index) {
        int leftIndex = 2 * index + 1;

        while (leftIndex < heapSize) {
            int rightIndex = leftIndex + 1;
            int greatestIndex = rightIndex < heapSize && this.data[rightIndex] > this.data[leftIndex] ? rightIndex : leftIndex;

            if (this.data[greatestIndex] <= this.data[index]) {
                break;
            }

            swap(index, greatestIndex);
            index = greatestIndex;
            leftIndex = 2 * index + 1;
        }
    }

    @Override
    public String toString() {
        StringBuilder b = new StringBuilder();
        b.append('[');
        for (int i = 0; i<heapSize; i++) {
            b.append(data[i]);
            if (i != heapSize - 1)
                b.append(", ");
        }
        b.append(']');

        return b.toString();
    }
}
```

输出内容：

```
[13, 9, 5, 3, 9, -4, -1, -6, -2, 0, 7]
13
[9, 9, 5, 3, 7, -4, -1, -6, -2, 0]
9
[9, 7, 5, 3, 0, -4, -1, -6, -2]
9
[7, 3, 5, -2, 0, -4, -1, -6]
7
[5, 3, -1, -2, 0, -4, -6]
5
[3, 0, -1, -2, -6, -4]
3
[0, -2, -1, -4, -6]
0
[-1, -2, -6, -4]
-1
[-2, -4, -6]
-2
[-4, -6]
-4
[-6]
-6
[]
```

* Rust 实现

```rust
pub struct Heap<T>(Vec<T>);

impl<T: Ord> Heap<T> {
    pub fn new() -> Self {
        Self(Vec::new())
    }

    pub fn push(&mut self, value: T) {
        self.0.push(value);
        self.heap_insert();
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.0.is_empty() {
            return None;
        }

        let last = self.0.len() - 1;
        self.0.swap(0, last);
        let result = self.0.pop();
        self.heapify();
        result
    }

    fn heapify(&mut self) {
        let len = self.0.len();

        let mut index = 0;
        let mut left = index * 2 + 1;

        while left < len {
            let right = left + 1;
            let greatest = if right < len && self.0[left] < self.0[right] {
                right
            } else {
                left
            };

            if self.0[greatest] <= self.0[index] {
                break;
            }

            self.0.swap(index, greatest);
            index = greatest;
            left = index * 2 + 1;
        }
    }

    fn heap_insert(&mut self) {
        if self.0.is_empty() {
            return;
        }

        let mut index = self.0.len() - 1;
        if index == 0 {
            return;
        }
        let mut parent = (index - 1) / 2;

        while self.0[parent] < self.0[index] {
            self.0.swap(index, parent);
            index = parent;
            if index == 0 {
                break;
            }
            parent = (index - 1) / 2;
        }
    }
}

impl<T: Ord> Iterator for Heap<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.pop()
    }
}

#[test]
fn test_heap() {
    for _ in 0..1000 {
        let array: [u8; 32] = rand::random();
        let sorted = {
            let mut s = array.clone();
            s.sort_by(|a, b| b.cmp(a));
            s
        };

        let mut heap = Heap::new();

        for a in array {
            heap.push(a);
        }

        let heap_sorted: Vec<u8> = heap.collect();

        assert_eq!(heap_sorted, sorted);
    }
}
```