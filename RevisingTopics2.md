# 🚀 Advanced Java Concurrency: Lock-Free, Coordination & Thread Pooling

This document covers advanced coordination primitives, lock-free data structures, the Executor framework, and critical production failure modes covered during our advanced concurrency session.

---

## 🏎️ 1. Atomics & Lock-Free Architecture (`CAS`)
* **The Concept:** Traditional locks require Operating System kernel intervention to pause and resume threads, which introduces heavy performance overhead. Atomics use **Compare-And-Swap (CAS)**, a hardware-level single CPU instruction.
* **The Formula:** It reads a value, computes a mutation, and updates memory *only if* the value hasn't changed since it read it. If another thread snuck in and modified the state, the CAS returns `false`, and the calling thread loops back to retry.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class GameScore {
    private final AtomicInteger score = new AtomicInteger(10);

    public void tripleScore() {
        int prev, next;
        do {
            prev = score.get(); // 1. Read the state
            next = prev * 3;    // 2. Compute mutation
        } while (!score.compareAndSet(prev, next)); // 3. Retry loop if overtaken
    }
}
```

---

## 🚦 2. Counted Permits (`Semaphore`)
* **The Concept:** Concurrency throttle that limits access to a resource pool using **permits**. Multiple threads can enter simultaneously up to the maximum number of permits allowed.
* **The Non-Reentrant Trap:** Unlike `synchronized` or `ReentrantLock`, a `Semaphore` is **not reentrant**. If a single thread attempts to acquire a permit twice from a `Semaphore(1)`, it will block itself indefinitely, resulting in a self-induced deadlock.

```java
import java.util.concurrent.Semaphore;

public class ApiGateway {
    private final Semaphore bouncer = new Semaphore(2); // Caps concurrent downloads to 2

    public void downloadFile() throws InterruptedException {
        bouncer.acquire();
        try {
            System.out.println("Downloading chunk...");
        } finally {
            bouncer.release(); // Must return permit inside finally block
        }
    }
}
```

---

## ⏳ 3. Latches vs. Barriers


| Primitives | Core Synchronization Pattern | Lifecycle | Blocking Behavior |
| :--- | :--- | :--- | :--- |
| **`CountDownLatch`** | One thread waits for $N$ separate threads to finish an action. | **One-Shot:** Once zeroed out, it cannot be reset. | `.countDown()` is non-blocking; `.await()` blocks the coordinator thread. |
| **`CyclicBarrier`** | $N$ parallel threads block and wait for each other to arrive at a checkpoint. | **Reusable:** Automatically resets once the group completes a phase. | All threads block via `.await()` until the group is fully gathered. |

---

## 📦 4. Thread-Safe Data Structures (`ConcurrentHashMap`)
* **The Compound Operation Trap:** While individual operations like `.containsKey()` and `.put()` on a `ConcurrentHashMap` are atomic, chaining them together sequentially breaks atomicity and introduces serious race conditions.
* **The Solution:** Use atomic compound primitives like `.putIfAbsent()` or `.computeIfAbsent()` to execute check-and-write mechanics in a single uninterrupted operation.

```java
import java.util.concurrent.ConcurrentHashMap;

public class UserManager {
    private final ConcurrentHashMap<String, String> userMap = new ConcurrentHashMap<>();

    public void registerUser(String username, String profileData) {
        // Safe & Atomic: Eliminates race conditions between checking and inserting
        userMap.putIfAbsent(username, profileData); 
    }
}
```

---

## 🏗️ 5. Resource Pooling & Scale Footguns (`ExecutorService`)
* **The Concept:** Instantiating raw operating system threads via `new Thread()` is highly resource-intensive. The Executor Framework maintains a persistent pool of reusable threads to handle arriving tasks.
* **The Production Out-of-Memory (OOM) Trap:** `Executors.newFixedThreadPool(n)` utilizes a completely **Unbounded LinkedBlockingQueue** under the hood. During an unexpected high-traffic spike, millions of backlog tasks will pile up inside this queue without limit, exhausting the JVM heap memory and crashing the application server with an `OutOfMemoryError`.
* **Production Best Practice:** Never use standard factory methods blindly. Always construct your own `ThreadPoolExecutor` explicitly, defining an isolated bounded queue size and a clear task rejection policy (e.g., `AbortPolicy` or `CallerRunsPolicy`).

---

## 🩺 6. Thread Interruption Etiquette
* **The Concept:** When a thread throws an `InterruptedException` (from `Thread.sleep`, `wait`, etc.), the JVM automatically **clears the interrupted status flag** of that thread.
* **The Swallowing Poison:** Catching the exception and leaving the catch block empty is a critical architectural bug. It erases the stop signal, meaning parent frameworks or loop structures lose all visibility that a shutdown was requested.
* **The Correction:** You must always either bubble up and rethrow the exception, or re-set the flag on the current thread using `Thread.currentThread().interrupt()` before exiting cleanly.

```java
public class LazyWorker implements Runnable {
    public void run() {
        while (true) {
            try {
                Thread.sleep(5000); 
            } catch (InterruptedException e) {
                // Correct: Restore the flag so downstream/upstream processes are notified
                Thread.currentThread().interrupt(); 
                return; // Clean exit from execution loop
            }
        }
    }
}
```
