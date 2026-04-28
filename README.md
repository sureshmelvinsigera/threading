# Threads vs Processes vs Async

Your app freezes. CPU is fine. Memory is fine. Nothing crashed. You've seen this before. Maybe in production. Maybe at 9:30am when the market opens. Today we find out exactly why — and how to fix it.

#### Learning Objectives

By the end of this session, you will be able to:

- Explain what a **process** is and why every Java program starts with one
- Describe how **threads** share memory inside a process — and why that's both powerful and dangerous
- Spot a **race condition**, explain exactly why it happens, and fix it with `synchronized`
- Explain why **thread pools** exist and why production Java never creates raw threads
- Fire concurrent operations with `CompletableFuture` and prove the difference — **3 seconds vs 1 second**
- Understand why **async doesn't mean parallel** — it means **non-blocking**

#### Step 1 — The Process and Its Threads `0:05 – 0:12`

```
┌─────────────────────────────────────────┐
│                 Process                 │
│            PID: 91265                   │
│                                         │
│  ┌──────────────┐  ┌──────┐  ┌──────┐   │
│  │  main thread │  │  T1  │  │  T2  │   │
│  │   (ID: 1)    │  │      │  │      │   │
│  └──────────────┘  └──────┘  └──────┘   │
│                                         │
│       shared memory / shared counter    │
└─────────────────────────────────────────┘
```

When you run a Java program, the OS creates a **process** — a fully isolated environment with its own **memory**, its own **file handles**, and a unique identifier called a **PID**.

The process doesn't do work on its own. That's the **thread's** job. Java starts one automatically — the **main thread** — before your first line of code runs.

```java
System.out.println("Hello World!");
System.out.println("Process ID : " + ProcessHandle.current().pid());
System.out.println("Main thread: " + Thread.currentThread().getName());
System.out.println("Main ID    : " + Thread.currentThread().threadId());
```

Open a second terminal and type `jps` — you'll see the **exact same PID**. The process is real. It's living in the OS right now.

#### Threads — Multiple Workers, Shared Memory

```java
static int counter = 0;

Runnable task = () -> {
    counter++;
    System.out.println("Current Thread: " + Thread.currentThread().getName());
};

Thread t1 = new Thread(task, "T1");
Thread t2 = new Thread(task, "T2");

t1.start();
t2.start();

System.out.println("counter: " + counter);
```

Run it a few times. Watch the **output order change**. T2 sometimes prints before T1 — even though T1 started first. That's the **OS scheduler**. You don't control it. Nobody does.

#### join() — Main Thread Waits

```java
t1.start();
t2.start();
t1.join(); // main waits — "I'm not moving until you finish"
t2.join();

System.out.println("counter: " + counter);
```

Without `join()` — main prints counter before T1 and T2 finish. With `join()` — main waits. The result is correct.

#### Step 2 — The Race Condition `0:18 – 0:25`

```java
Runnable task = () -> {
    for (int i = 0; i < 10_000; i++) {
        counter++; // READ → ADD → WRITE
    }
};

for (int run = 1; run <= 10; run++) {
    counter = 0;

    Thread r1 = new Thread(task, "Thread T1");
    Thread r2 = new Thread(task, "Thread T2");

    r1.start();
    r2.start();
    r1.join();
    r2.join();

    System.out.println("Run " + run + " → Expected: 20000, Got: " + counter);
}
```

`counter++` looks like **one operation**. It's actually **three**:

```
Chef T1 reads counter  — sees 5
Chef T2 reads counter  — sees 5   ← grabbed it before T1 wrote back
Chef T1 adds 1         — now 6
Chef T2 adds 1         — now 6
Chef T1 writes 6
Chef T2 writes 6                   ← overwrites T1's work
```

Counter should be 7. We got 6. **One count lost. Nobody noticed.**

Ten runs. Ten different wrong answers. This is **non-determinism** — not a one-off bug. Threads don't fail loudly. They **fail silently**.

```
Run 1  → Expected: 20000, Got: 13405
Run 2  → Expected: 20000, Got: 14823
Run 3  → Expected: 20000, Got: 11907
Run 4  → Expected: 20000, Got: 15488
Run 5  → Expected: 20000, Got: 13220
Run 6  → Expected: 20000, Got: 16004
Run 7  → Expected: 20000, Got: 12738
Run 8  → Expected: 20000, Got: 14391
Run 9  → Expected: 20000, Got: 13856
Run 10 → Expected: 20000, Got: 15102
```

#### Step 3 — The Fix `0:25 – 0:30`

```java
static synchronized void increment() {
    counter++;
}

Runnable task = () -> {
    for (int i = 0; i < 10_000; i++) {
        increment();
    }
};
```

`synchronized` enforces **mutual exclusion** — only one thread executes `increment()` at a time. The other waits. Every Java object has an **intrinsic lock** — `synchronized` uses that lock.

Run it 10 times. **Always 20,000**. Every single time.

```
Run 1  → Expected: 20000, Got: 20000
Run 2  → Expected: 20000, Got: 20000
...
Run 10 → Expected: 20000, Got: 20000
```

The trade-off: **contention**. Blocked threads waiting. Use `synchronized` as **narrowly as possible**.

#### Thread Pools — Reuse, Don't Rebuild `0:30 – 0:35`

Creating a thread has **overhead** — memory allocation, OS registration, setup, teardown. For applications handling many short-lived jobs, creating a new thread per task is wasteful.

A **thread pool** creates a fixed set of threads once and **reuses** them across many tasks.

```java
ExecutorService pool = Executors.newFixedThreadPool(3);

for (int i = 1; i <= 6; i++) {
    final int taskId = i;
    pool.submit(() -> {
        System.out.println("Task " + taskId + " on: "
            + Thread.currentThread().getName());
    });
}

pool.shutdown();
boolean finished = pool.awaitTermination(10, TimeUnit.SECONDS);
System.out.println("All tasks finished: " + finished);
```

`pool-1-thread-1` handles task 1, then comes back for task 4. **No new thread created.** Same three threads. Six tasks. That's **reuse**.

In production Java — you almost never write `new Thread()` directly. **`ExecutorService` is the standard.**

---

### Async — Stop Waiting, Start Working `0:35 – 0:43`

Thread pools solve **reuse**. But there's still a problem — what happens when a thread **spends most of its time waiting**?

A thread making a network call sits **idle** for most of that operation. Under load, a server can exhaust all its threads just waiting — with nothing left to handle new requests.

**Key distinction: async doesn't mean parallel. It means non-blocking.**

#### Blocking — sequential, main thread waits for each call

```java
long start = System.currentTimeMillis();

String price1 = fetchStockPrice("AAPL"); // waits ~1 second
String price2 = fetchStockPrice("GOOG"); // waits ~1 second
String price3 = fetchStockPrice("MSFT"); // waits ~1 second

System.out.println("Blocking: " + (System.currentTimeMillis() - start) + "ms");
```

```
Fetching AAPL on thread: main
Fetching GOOG on thread: main
Fetching MSFT on thread: main
Blocking: 3068ms
```

#### Async — all three fire at the same time

```java
start = System.currentTimeMillis();

CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> fetchStockPrice("AAPL"));
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> fetchStockPrice("GOOG"));
CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> fetchStockPrice("MSFT"));

price1 = f1.get();
price2 = f2.get();
price3 = f3.get();

System.out.println("Async: " + (System.currentTimeMillis() - start) + "ms");
```

```
Fetching AAPL on thread: ForkJoinPool.commonPool-worker-1
Fetching GOOG on thread: ForkJoinPool.commonPool-worker-2
Fetching MSFT on thread: ForkJoinPool.commonPool-worker-3
Async: 1009ms
```

> **Same work. One second. We didn't make it faster. We just stopped wasting time waiting.**

`CompletableFuture` is an **IOU** — fire the work, move on, collect the result when ready. By default it runs on the **common ForkJoinPool** — a shared pool the JVM manages for you.

#### Choosing the Right Tool `0:43 – 0:44`

| Work Type | Model | Why |
|---|---|---|
| **CPU-bound** | More threads | Runs in **parallel** across multiple cores |
| **I/O-bound** | Async | **Non-blocking** — don't waste threads waiting |

**CPU-bound — add more threads. I/O-bound — go async.**

#### Summary + Q&A `0:44 – 0:45`

- A **process** is a fully isolated running instance of a program — its own **memory**, its own **PID**. One process crashing cannot affect another.

- **Threads** share memory inside a process. Fast — but dangerous. Two threads writing to the same variable at the same time produce a **race condition** — silent, inconsistent, nearly impossible to reproduce in a debugger.

- `synchronized` ensures **only one thread** executes at a time. Every Java object has an intrinsic lock. `synchronized` uses it. The cost is **contention**. Apply it **narrowly**.

- A **thread pool** reuses a fixed set of threads. **Creation cost** paid once. **Reuse** for everything. `ExecutorService` is the standard.

- **Async doesn't mean parallel. It means non-blocking.** `CompletableFuture` — an IOU — fires operations without blocking. Three calls — **3 seconds blocking, 1 second async**. Same work. We just stopped wasting time waiting.

- **Processes protect you. Threads speed you up. Async keeps you from waiting.**
