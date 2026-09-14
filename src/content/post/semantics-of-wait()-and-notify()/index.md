---
title: "Semantics of wait() and notify()"
description: "A closer look at how wait() and notify() work with Java monitors, and what actually happens to a waiting thread when notify() is called."
publishDate: "14 Sept 2026"
updatedDate: "14 Sept 2026"
draft: true
---
<hr />

When a thread calls `wait()`, we usually says thread goes to sleep. </br>
When another thread calls `notify()`, we usually say that waiting thread is woken up. </br>
That explanation is convenient, but it hides a few important details.

What does it actually mean for a thread to “wait”? What happens to the monitor while it is waiting? And when notify() is
called, does the waiting thread immediately start running?

These questions become interesting when multiple threads are waiting on the same object.

To understand this, let's look at a small blocking queue implementation using Java's `wait()` and `notify()`.

The queue has a fixed capacity. When it is full, a producer waits. When an element is removed, the consumer calls `notify()`.

At first, the behavior seems straightforward.

But with two producers waiting at the same time, something interesting happens: the thread that continues after notify() is not necessarily the thread that started waiting first.

That behavior is a good starting point for understanding what wait() and notify() actually guarantee—and what they don't.
<hr />

