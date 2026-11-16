# Parallel and GPU Computing Lab — Experiment 5: Analyze Synchronization Mechanisms in Parallel Programs

## 1. Experiment Overview & Objectives

In shared-memory parallel programming models such as OpenMP and POSIX Threads (Pthreads), concurrent execution of threads accessing shared data structures can introduce non-deterministic race conditions and data corruption. Proper synchronization primitives are required to enforce mutual exclusion, ensure memory consistency, and coordinate phase execution across parallel threads.

### Objectives
1. **Analyze Race Conditions**: Demonstrate how concurrent, unsynchronized updates to a shared variable lead to lost updates and non-deterministic results.
2. **Implement Mutual Exclusion Primitives**:
   - Utilize OpenMP Critical Sections (`#pragma omp critical`) to enforce mutual exclusion at the block level.
   - Utilize OpenMP Atomic Operations (`#pragma omp atomic`) for lightweight, hardware-assisted memory location updates.
   - Utilize POSIX Semaphores (`sem_t`, `sem_wait`, `sem_post`) to coordinate thread access in standard POSIX threading environments.
3. **Analyze Phase Synchronization**: Implement OpenMP Barriers (`#pragma omp barrier`) to synchronize thread execution across distinct computational phases.
4. **Empirical Benchmarking & Scalability Analysis**: Systematically evaluate the performance, runtime overhead, and scalability of `None` (unsynchronized), `Critical`, `Atomic`, and `Lock` across varying thread counts ($1, 2, 4, 8$) for $10,000,000$ shared operations.

---

## 2. Experimental Environment & Specifications

The experiment was conducted and verified in the following hardware and software environment:

| Specification | Details |
| :--- | :--- |
| **Operating System** | Ubuntu 26.04.1 LTS |
| **Compiler** | GCC 15.2.0 (`gcc`) |
| **Parallel Standards** | OpenMP (`-fopenmp`), POSIX Threads (`-pthread`) |
| **Available Processors** | 16 Logical Processors |
| **Benchmark Workload** | $10^7$ ($10,000,000$) shared increments per configuration |

---

## 3. Project Structure

```text
Experiment_5_Synchronization/
├── .gitignore                      # Git configuration ignoring binaries and OS files
├── README.md                       # Comprehensive lab documentation & analysis
├── race.c                          # Demonstrates unsynchronized race conditions
├── critical.c                      # Enforces mutual exclusion via OpenMP critical directive
├── atomic.c                        # Enforces mutual exclusion via OpenMP atomic directive
├── barrier.c                       # Demonstrates thread barrier synchronization
├── semaphore.c                     # Implements mutual exclusion using POSIX semaphores
├── compare_sync.c                  # Comparative benchmark across 1, 2, 4, and 8 threads
├── results/
│   └── benchmark_run1.txt          # Raw output from compare_sync execution
└── screenshots/
    ├── 01_race_condition.png       # Execution screenshot of race.c
    ├── 02_critical_section.png     # Execution screenshot of critical.c
    ├── 03_atomic_operation.png     # Execution screenshot of atomic.c
    ├── 04_barrier.png              # Execution screenshot of barrier.c
    ├── 05_semaphore.png            # Execution screenshot of semaphore.c
    └── 06_benchmark.png            # Execution screenshot of compare_sync.c
```

---

## 4. Source Programs Description

### 4.1. Race Condition (`race.c`)
- **Mechanism**: Unsynchronized concurrent access.
- **Implementation**: Executes a loop of $10,000,000$ iterations distributed among threads using `#pragma omp parallel for`. In each iteration, `counter++` is executed without protection.
- **Behavior**: Incrementing a variable (`counter++`) involves three distinct machine-level steps:
  1. *Read*: Load value from memory into a CPU register.
  2. *Modify*: Increment register value by 1.
  3. *Write*: Store updated register value back to memory.
  When multiple threads interleave these operations, simultaneous read-modify-write sequences overwrite each other, causing massive lost updates. The final count falls far short of $10,000,000$.

### 4.2. Critical Section (`critical.c`)
- **Mechanism**: Coarse-grained structured mutual exclusion via `#pragma omp critical`.
- **Implementation**: Encloses `counter++` inside `#pragma omp critical`.
- **Behavior**: Ensures that only one thread executes the enclosed structured block at any given instant. While correctness is fully preserved (Final Count = $10,000,000$), threads suffer significant queueing delays and lock contention overhead as all iterations are serialized at the lock entry.

### 4.3. Atomic Operation (`atomic.c`)
- **Mechanism**: Hardware-supported fine-grained memory update via `#pragma omp atomic`.
- **Implementation**: Prefaces `counter++` with `#pragma omp atomic`.
- **Behavior**: Instructs the compiler to generate hardware-supported atomic read-modify-write instructions (e.g., `LOCK XADD` on x86-64). Because it locks only the memory location for the single instruction rather than acquiring a full software lock object, it provides high throughput and superior multi-core scalability while guaranteeing a correct count of $10,000,000$.

### 4.4. Barrier Synchronization (`barrier.c`)
- **Mechanism**: Phase synchronization via `#pragma omp barrier`.
- **Implementation**: Within a `#pragma omp parallel` team, threads execute an initial print statement, encounter `#pragma omp barrier`, and then proceed to a second print statement.
- **Behavior**: Acts as an execution checkpoint. No thread in the parallel team can proceed past the barrier until all threads have reached it, preventing race conditions between sequential computational phases.

### 4.5. POSIX Semaphore (`semaphore.c`)
- **Mechanism**: OS/Pthreads-level binary semaphore (`sem_t`).
- **Implementation**: Spawns two threads (`t1`, `t2`) using `pthread_create()`. Each thread executes $100,000$ loop increments guarded by `sem_wait(&semaphore)` and `sem_post(&semaphore)`.
- **Behavior**: Initialized with value 1 (`sem_init(&semaphore, 0, 1)`), functioning as a binary semaphore (mutex). Both threads strictly alternate through critical section entry, yielding a perfectly synchronized final count of $200,000$.

### 4.6. Comparative Synchronization Benchmark (`compare_sync.c`)
- **Mechanism**: Comprehensive harness evaluating:
  1. **None**: No synchronization (Race condition).
  2. **Critical**: `#pragma omp critical`.
  3. **Atomic**: `#pragma omp atomic`.
  4. **Lock**: OpenMP explicit runtime lock API (`omp_lock_t`, `omp_set_lock()`, `omp_unset_lock()`).
- **Implementation**: Measures execution runtime using `omp_get_wtime()` for each mechanism across $1, 2, 4,$ and $8$ threads performing $10,000,000$ shared increments.

---

## 5. Compilation and Execution Guide

All programs can be compiled using GCC on Linux. Ensure `build-essential` and OpenMP support are available.

### 5.1. Compilation Commands

```bash
# 1. Race condition demonstration
gcc -fopenmp race.c -o race

# 2. Critical section demonstration
gcc -fopenmp critical.c -o critical

# 3. Atomic operation demonstration
gcc -fopenmp atomic.c -o atomic

# 4. Barrier synchronization demonstration
gcc -fopenmp barrier.c -o barrier

# 5. POSIX Semaphore demonstration
gcc -pthread semaphore.c -o semaphore

# 6. Synchronization comparative benchmark
gcc -fopenmp compare_sync.c -o compare_sync
```

### 5.2. Execution Commands

```bash
# Run individual programs
./race
./critical
./atomic
./barrier
./semaphore
./compare_sync

# To save benchmark output directly to a file:
./compare_sync > results/benchmark_run1.txt
```

---

## 6. Empirical Benchmark Results

The following data was empirically obtained on Ubuntu 26.04.1 LTS with 16 logical processors (recorded in `results/benchmark_run1.txt`):

```text
Comparing synchronization mechanisms for 10000000 shared increments

Mechanism | Threads | Final Count | Correctness |     Time (s)
---------------------------------------------------------------
None       |       1 |     10000000 |    Correct |     0.0084
Critical   |       1 |     10000000 |    Correct |     0.2941
Atomic     |       1 |     10000000 |    Correct |     0.1192
Lock       |       1 |     10000000 |    Correct |     0.2387
---------------------------------------------------------------
None       |       2 |      5461304 |  INCORRECT |     0.0160
Critical   |       2 |     10000000 |    Correct |     0.4563
Atomic     |       2 |     10000000 |    Correct |     0.1772
Lock       |       2 |     10000000 |    Correct |     0.4917
---------------------------------------------------------------
None       |       4 |      3795132 |  INCORRECT |     0.0247
Critical   |       4 |     10000000 |    Correct |     1.0539
Atomic     |       4 |     10000000 |    Correct |     0.2930
Lock       |       4 |     10000000 |    Correct |     1.5438
---------------------------------------------------------------
None       |       8 |      2193073 |  INCORRECT |     0.0290
Critical   |       8 |     10000000 |    Correct |     1.6518
Atomic     |       8 |     10000000 |    Correct |     0.3522
Lock       |       8 |     10000000 |    Correct |     3.0210
---------------------------------------------------------------

Expected correct final count = 10000000
```

### Consolidated Performance Comparison Table

| Mechanism | Threads | Final Count | Accuracy (%) | Execution Time (s) | Overhead vs None (1T) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **None** | 1 | 10,000,000 | 100.0% | **0.0084 s** | $1.0\times$ (Baseline) |
| **None** | 2 | 5,461,304 | 54.61% | 0.0160 s | — |
| **None** | 4 | 3,795,132 | 37.95% | 0.0247 s | — |
| **None** | 8 | 2,193,073 | 21.93% | 0.0290 s | — |
| **Critical** | 1 | 10,000,000 | 100.0% | 0.2941 s | $35.0\times$ |
| **Critical** | 2 | 10,000,000 | 100.0% | 0.4563 s | $54.3\times$ |
| **Critical** | 4 | 10,000,000 | 100.0% | 1.0539 s | $125.5\times$ |
| **Critical** | 8 | 10,000,000 | 100.0% | 1.6518 s | $196.6\times$ |
| **Atomic** | 1 | 10,000,000 | 100.0% | **0.1192 s** | $14.2\times$ |
| **Atomic** | 2 | 10,000,000 | 100.0% | **0.1772 s** | $21.1\times$ |
| **Atomic** | 4 | 10,000,000 | 100.0% | **0.2930 s** | $34.9\times$ |
| **Atomic** | 8 | 10,000,000 | 100.0% | **0.3522 s** | **$41.9\times$** |
| **Lock** | 1 | 10,000,000 | 100.0% | 0.2387 s | $28.4\times$ |
| **Lock** | 2 | 10,000,000 | 100.0% | 0.4917 s | $58.5\times$ |
| **Lock** | 4 | 10,000,000 | 100.0% | 1.5438 s | $183.8\times$ |
| **Lock** | 8 | 10,000,000 | 100.0% | 3.0210 s | $359.6\times$ |

---

## 7. In-Depth Analysis: Correctness & Scalability

### 7.1. Correctness & Lost Updates in Unsynchronized Parallelism
- Under single-threaded execution ($T = 1$), `None` completes in $0.0084$ seconds and returns the correct result ($10,000,000$) because no concurrency exists.
- As the thread count increases, the data corruption escalates drastically:
  - **2 Threads**: Final count is $5,461,304$ (Lost updates: $45.39\%$).
  - **4 Threads**: Final count is $3,795,132$ (Lost updates: $62.05\%$).
  - **8 Threads**: Final count is $2,193,073$ (Lost updates: $78.07\%$).
- **Explanation**: In an unsynchronized parallel for loop, multiple threads access the same memory address simultaneously. When Thread $A$ reads `counter = 50`, Thread $B$ also reads `counter = 50`. Both increment their local register to `51` and write it back. Instead of two increments occurring, only one is registered, losing one update. As concurrency rises from 2 to 8 threads, the window for overlapping read-modify-write operations expands, causing up to ~78% data loss.

### 7.2. Performance & Scalability of Synchronization Mechanisms

```text
Execution Time vs. Thread Count (Lower is Better)
-----------------------------------------------------------------
Time (s)
3.5 |                                                [Lock: 3.02s]
3.0 |
2.5 |
2.0 |
1.5 |                                [Critical: 1.65s]
1.0 |                    [Lock: 1.54s]
0.5 |    [Lock: 0.49s]   [Critical: 1.05s]
    |    [Crit: 0.46s]   [Atomic: 0.29s]    [Atomic: 0.35s]
0.0 +----+---------------+------------------+---------------+----
         1 Thread        2 Threads          4 Threads       8 Threads
```

1. **Atomic (`#pragma omp atomic`) — The Optimal Choice**:
   - Runtime grows modestly from $0.1192$ s (1 thread) to only $0.3522$ s (8 threads).
   - Atomic operations avoid software lock table allocation, OS sleep/wake cycles, and function call overhead.
   - The CPU cache-coherency hardware manages the atomicity directly at the L1/L2 cache line level via bus-level arbitration or cache line locking (e.g., MESI cache protocol).
   - At 8 threads, **Atomic is $4.69\times$ faster than Critical and $8.58\times$ faster than Lock**.

2. **Critical (`#pragma omp critical`) — Moderate Contention**:
   - Runtime scales from $0.2941$ s (1 thread) up to $1.6518$ s (8 threads).
   - `critical` establishes a compiler-managed global lock around the block. As thread count increases, threads spend an overwhelming fraction of time waiting in spin/queue locks to acquire exclusive access, completely destroying parallel efficiency.

3. **Lock (`omp_lock_t`) — Severe Scalability Degradation**:
   - Runtime escalates from $0.2387$ s (1 thread) to $3.0210$ s (8 threads).
   - While explicit locks provide flexible non-block-structured locking, calling `omp_set_lock()` and `omp_unset_lock()` $10^7$ times introduces substantial library call overhead, atomic lock variable state thrashing, and cache-line bouncing across all active cores.
   - Consequently, explicit locks exhibited the worst scaling curve under extreme contention.

---

## 8. Verification Screenshots

All execution runs were verified and captured under `screenshots/`:

| No. | Description | Image Preview |
| :---: | :--- | :--- |
| **01** | **Race Condition Demonstration (`race.c`)**<br>Demonstrates incorrect counter value due to lost updates. | ![Race Condition](screenshots/01_race_condition.png) |
| **02** | **Critical Section (`critical.c`)**<br>Ensures mutual exclusion and produces the correct result ($10,000,000$). | ![Critical Section](screenshots/02_critical_section.png) |
| **03** | **Atomic Operation (`atomic.c`)**<br>Guarantees atomic memory update and correct count ($10,000,000$) with low overhead. | ![Atomic Operation](screenshots/03_atomic_operation.png) |
| **04** | **Barrier Synchronization (`barrier.c`)**<br>Synchronizes thread teams, ensuring distinct pre- and post-barrier output phases. | ![Barrier](screenshots/04_barrier.png) |
| **05** | **POSIX Semaphore Synchronization (`semaphore.c`)**<br>Coordinates two POSIX threads using a binary semaphore ($200,000$ count). | ![Semaphore](screenshots/05_semaphore.png) |
| **06** | **Comparative Benchmark (`compare_sync.c`)**<br>Empirical scalability comparison across 1, 2, 4, and 8 threads. | ![Benchmark](screenshots/06_benchmark.png) |

---

## 9. Key Observations & Conclusions

1. **Race Conditions are Silent & Catastrophic**:
   - Without synchronization, concurrent programs fail silently without crashes or compiler warnings, producing non-deterministic and corrupted data (up to $78.07\%$ data loss at 8 threads).

2. **Synchronization Incurs Contention Overhead**:
   - In shared-state benchmarks where all threads contend for the exact same memory address, adding more threads increases execution time rather than decreasing it. This experimentally demonstrates the cost of synchronization contention.

3. **Hardware-Assisted Atomics Outperform Software Locks**:
   - For simple operations (e.g., increments, decrements, bitwise operations), OpenMP `#pragma omp atomic` should always be preferred over `#pragma omp critical` or `omp_lock_t`.
   - In this benchmark, `atomic` delivered up to **$8.58\times$ higher throughput** than explicit locks under 8-thread contention while providing guaranteed correctness.

4. **Coarse Locks vs. Fine Locks**:
   - OpenMP critical sections provide a high-level abstraction for multi-statement mutual exclusion, while explicit OpenMP locks offer flexibility across functions. However, both impose significant performance penalties when applied in fine-grained, high-frequency loops.

5. **Phase Synchronization**:
   - OpenMP barriers provide an effective synchronization mechanism to coordinate algorithms with dependent stages, guaranteeing that no thread proceeds until all threads have completed the current phase.
