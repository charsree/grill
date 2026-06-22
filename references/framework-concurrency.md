# Concurrency / shared-state reviewer

Source: Lamport (happens-before, 1978); Herlihy & Shavit (The Art of Multiprocessor Programming); Goetz (Java Concurrency in Practice); the Rust ownership model; Facebook Infer's RacerD; Google's ThreadSanitizer methodology. The core insight: concurrency bugs are invisible in sequential reading because they require reasoning about INTERLEAVING, not about any single execution path.

## When this lens fires

This lens runs on ANY change that touches or introduces:
- Locks, mutexes, semaphores, RWLocks, atomic operations
- Async/await, futures, promises, channels, message queues
- Thread/goroutine/task spawning
- Shared mutable state (global variables, static mut, class-level fields accessed from multiple threads)
- Database transactions, optimistic locking, compare-and-swap
- Caches (read/write from multiple contexts)
- Event loops, signal handlers, interrupt handlers

## The 7 concurrency bug patterns

### 1. Data race / unprotected shared access

**What:** Two or more threads/tasks access the same mutable state, at least one is a write, with no synchronization between them.

**Check:** For every field/variable the diff reads or writes:
- Can another thread/task/handler access it?
- If yes, what synchronization protects it? (Lock, atomic, channel, ownership transfer, confinement to one thread.)
- Is the synchronization consistent? (Same lock for all accesses? Not sometimes-locked-sometimes-not?)

**Subtle variant:** A field protected by lock A is also accessed in a context that only holds lock B. Or: a field is atomic but the operation on it is not (read-modify-write of an AtomicI32 without compare_exchange).

### 2. Atomicity violation (check-then-act / TOCTOU)

**What:** A sequence of operations that must be atomic is not protected as a unit. Another actor can intervene between the check and the action.

**Check:**
- "if (condition) { act on condition }" where the condition can change between the check and the action.
- "read value, compute new value, write new value" without holding a lock or using CAS for the entire sequence.
- "check existence, then create" (another thread creates between your check and your create).
- File operations: check-then-open (TOCTOU classic).

**The tell:** any time you see a gap between observing state and acting on that observation where another actor could modify the state.

### 3. Ordering violation / happens-before

**What:** Code assumes an ordering between operations in different threads that is not enforced.

**Check:**
- "Thread A publishes data, Thread B reads it" without a synchronization point that establishes happens-before (a lock release/acquire, a channel send/receive, a barrier, a volatile/atomic with appropriate ordering).
- "Initialization in one thread, use in another" without ensuring init completes first.
- "Subscribe to events, then start producer" vs "start producer, then subscribe" (can miss events).
- Async operations assumed to complete in dispatch order (they may not).

### 4. Deadlock / lock ordering

**What:** Two or more locks acquired in different orders by different code paths, creating a cycle.

**Check:**
- Does the diff acquire a lock while already holding another? What is the order? Is that order consistent with all other code that holds both?
- Does the diff call into code that acquires a lock, while already holding a lock? (Recursive, transitive lock ordering.)
- Does the diff hold a lock across an await point? (In async code, this can deadlock if the executor is single-threaded, or if the awaited future needs the same lock.)

### 5. Starvation / livelock / unfairness

**What:** A thread/task can be indefinitely delayed or makes no progress despite not being deadlocked.

**Check:**
- A busy-wait (spin loop) that can spin forever if the condition is never met.
- A lock held for a long time (across I/O, across a network call) starving other waiters.
- Priority inversion: a high-priority task waiting on a low-priority task that is preempted.
- Try-lock loops without backoff.

### 6. Incorrect lifecycle / use-after-free in concurrent context

**What:** A resource (connection, handle, object) is closed/freed/dropped while another thread still holds a reference and may use it.

**Check:**
- "Shutdown" or "close" methods that invalidate state while concurrent operations are in flight.
- Removing an item from a shared collection while another thread iterates or holds a reference.
- Cancellation that does not wait for in-flight operations to complete.
- Object pools where an object is returned to the pool and reused while a previous borrower still holds it.

### 7. Lost wakeup / missed signal

**What:** A notification (condition variable signal, channel send, event) is sent before the receiver is waiting, so it is lost.

**Check:**
- Condition variable signal without holding the associated lock (receiver can miss it).
- Channel send before the receiver is listening, in a non-buffered channel.
- "Set flag, then notify" vs "notify, then set flag" (receiver checks flag, sees old value, goes back to sleep).
- Spurious wakeup not handled (while-loop around condition wait vs if-check).

## Review method

For each piece of shared state the diff touches:

1. **Identify all accessors.** Who reads it? Who writes it? From which threads/tasks/contexts?
2. **Identify the synchronization.** What mechanism ensures mutual exclusion or ordering?
3. **Check completeness.** Is EVERY access protected? Not just the one in the diff, but ALL accesses you can find?
4. **Check the critical section scope.** Is the protection held for the right duration? (Too short: another operation between steps. Too long: deadlock risk, performance.)
5. **Check the interleaving.** Mentally insert "another thread runs here" between every pair of operations. Does anything break?

## How a finding should read

Name the shared state, the two (or more) concurrent actors, the interleaving that causes the bug, and what breaks. "The `connectionPool` map is read at line 45 (in the request handler, on the HTTP thread) and modified at line 112 (in the health-check goroutine). No lock protects the map. If the health-check removes a stale connection between the handler's `get()` and its use of the connection, the handler uses a closed connection. Under load this is a data race (go race detector would flag it) and can panic or return garbage."
