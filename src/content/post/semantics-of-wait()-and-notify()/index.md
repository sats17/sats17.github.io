---
title: "Semantics of wait() and notify()"
description: "A closer look at how wait() and notify() work with Java monitors, and what actually happens to a waiting thread when notify() is called."
publishDate: "14 Sept 2026"
updatedDate: "14 Sept 2026"
draft: true
---
<hr />
When a thread calls `wait()`, we usually say that the thread goes to sleep.

When another thread calls `notify()`, we usually say that the waiting thread is woken up.

That explanation is convenient, but it hides a few important details.

What does it actually mean for a thread to “wait”? What happens to the monitor while it is waiting? And when `notify()` is called, does the waiting thread immediately start running?

These questions become interesting when multiple threads are waiting on the same object.

To understand this, let's look at a small blocking queue implementation using Java's `wait()` and `notify()`.

The queue has a fixed capacity. When the queue is full, a producer waits. When an element is removed, the consumer calls `notify()`.

At first, the behavior seems straightforward.

But with two producers waiting at the same time, something interesting happens: the thread that continues after `notify()` is not necessarily the thread that started waiting first.

<hr/>

Let's start with a small blocking queue implemented using a circular array. The queue uses `putIndex` to determine where the next element should be inserted and `takeIndex` to determine where the next element should be removed.


```java
    public synchronized void put(E data) throws Exception {
        while(isQueueFull(putIndex)) {
            System.out.println("Queue is full, waiting for any space available");
            wait();
            System.out.println("Wait complete");
        }
        arr[putIndex] = data;
        putIndex = nextIndex(putIndex);
    }
```
The `put()` method is `synchronized`, which means the thread executing it must first acquire the monitor of the queue object.

While the queue is full, the producer cannot insert an element, so it calls `wait()`.

Calling `wait()` does two important things:

1. The thread enters the `WAITING` state.
2. It releases the monitor of the queue object.

Releasing the monitor is important here. Without it, another thread would not be able to enter a synchronized method such as `take()` and make space in the queue.

Another producer can now acquire the same monitor and enter `put()`. If the queue is still full, that producer also calls `wait()`.

At this point, we can have multiple threads waiting on the same queue object.
