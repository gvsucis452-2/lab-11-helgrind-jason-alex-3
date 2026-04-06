# CIS 452 Lab — Helgrind

**Names:** Alex Tillery, Jason Gray-Moore  
**Professor:** Dr. Kurmas  
**Course:** CIS 452 03  
**Lab:** 11  
**Term:** Winter 2026  

---

## Overview

This lab focuses on using **Helgrind**, a Valgrind tool for detecting synchronization problems in multi-threaded programs. The purpose of the lab is to observe how Helgrind reports **data races**, **deadlocks**, and **thread signaling issues**, and then compare the tool’s reports to what is actually happening in the code.

---

## Part A — `main-race.c`

### 1.

**What is the obvious problem in `main-race.c`?**

An issue with the program is that both the main thread and the worker thread could access and modify the shared global variable. Each thread does a balance++ which has multiple steps internal and could be interrupted during one of the steps.

### 2.

**Run:**

```bash
valgrind --tool=helgrind ./main-race
```

**Does Helgrind point to the correct lines of code? What other information does it give you?**

Yes, Helgrind points to the correct lines where `balance++` is accessed in both the main thread and the worker thread. It also shows which threads are involved, the type of access (read/write), and that the shared variable `balance` is the source of the conflict. It includes stack traces for both accesses as well.


### 3.

**What happens when you remove one of the offending lines of code?**

After commenting out one `balance++` line, Helgrind reported 0 errors. This is because only one thread was updating the shared variable.

### 4.

**What happens when you add a lock around only one of the updates to the shared variable? What does Helgrind report?**

The following code was added in the worker function.

```c
pthread_mutex_lock(&m);
balance++;
pthread_mutex_unlock(&m);
```

Helgrind still reports a data race because only one thread is protected.

### 5.

**What happens when you add locks around both updates? What does Helgrind report?**

The following code was added in the main function.

```c
pthread_mutex_lock(&m);
balance++;
pthread_mutex_unlock(&m);
```

Helgrind reports 0 errors because both threads are properly synchronized.

---

## Part B — `main-deadlock.c`

### 6.

**Describe what a deadlock is. Why specifically does `main-deadlock.c` have a deadlock?**

A deadlock is when two threads are each waiting for a resource that the other thread is holding, so neither one can continue. In `main-deadlock.c`, the problem is that the two threads lock the same mutexes in opposite order. One thread locks `m1` then `m2`, while the other locks `m2` then `m1`. That creates a deadlock risk because each thread can end up waiting for the other.

### 7.

**Run Helgrind on `main-deadlock.c`. What does it report?**

```bash
valgrind --tool=helgrind ./main-deadlock
./main-deadlock
```

Helgrind reports a lock order violation. It says the required order is `m1` before `m2`, but one thread acquires them in the opposite order. That matches the actual problem in the code. When I ran `./main-deadlock` normally, it exited right away and did not visibly hang, but the code still has a deadlock risk because deadlocks depend on timing and may not happen every run.

---

## Part C — `main-deadlock-global.c`

### 8.

**Does `main-deadlock-global.c` have the same problem that `main-deadlock.c` has? Why or why not? Should Helgrind be reporting the same error? What does this tell you about tools like Helgrind?**

`main-deadlock-global.c` still shows the same lock ordering issue to Helgrind. Helgrind reports a lock order violation again, even though running the program normally did not make it visibly hang. This suggests that the code still has the same suspicious locking pattern involving `m1` and `m2`. It also shows that tools like Helgrind are useful for finding possible synchronization problems, but the results still need to be interpreted carefully by a person. A warning does not always mean the program will visibly fail every run.

---

## Part D — `main-signal.c`

### 9.

**Why is this code inefficient? What does the parent spend its time doing, especially if the child takes a long time to complete?**

This code is inefficient because the parent is busy waiting. It repeatedly checks the shared variable `done` in a loop instead of sleeping or blocking. If the child takes a long time to complete, the parent spends its time continuously checking `done` and wasting CPU time.

### 10.

**Run Helgrind on `main-signal.c`. What does it report? Is the code correct?**

```bash
valgrind --tool=helgrind ./main-signal
./main-signal
```

Helgrind reports a possible data race on the shared variable `done`. The main thread reads `done` while the worker thread writes to it, and neither access is protected by a lock. The program output appears correct because it prints `this should print first` and then `this should print last`, but the code is still not properly synchronized. So it may seem to work, but it is not a correct threaded solution.

---

## Part E — `main-signal-cv.c`

### 11.

**Why is this code preferred over the previous version? Is the improvement related to correctness, performance, or both?**

This version is preferred because it uses proper synchronization with a mutex and condition variable instead of busy waiting. It is better for performance because the waiting thread can sleep instead of constantly checking a variable. It is also better for correctness because the shared state is synchronized properly. So the improvement is related to both correctness and performance.

### 12.

**Run Helgrind on `main-signal-cv.c`. Does it report any errors?**

```bash
valgrind --tool=helgrind ./main-signal-cv
./main-signal-cv
```

Helgrind reports 0 errors for `main-signal-cv.c`. The program output is also correct, with `this should print first` appearing before `this should print last`. This shows that the condition variable version is properly synchronized.

---

## Commands Used

```bash
make
valgrind --tool=helgrind ./main-race
make main-race
valgrind --tool=helgrind ./main-race
make main-race
valgrind --tool=helgrind ./main-race
make
valgrind --tool=helgrind ./main-race
valgrind --tool=helgrind ./main-deadlock
./main-deadlock
valgrind --tool=helgrind ./main-deadlock-global
./main-deadlock-global
valgrind --tool=helgrind ./main-signal
./main-signal
valgrind --tool=helgrind ./main-signal-cv
./main-signal-cv
```

---

## Observations / Results

### `main-race.c`
- Helgrind reported a data race on the shared variable `balance`
- It pointed to the exact lines where both threads accessed `balance++`
- Removing one update removed the race completely
- Locking only one update did not fix the problem
- Locking both updates with the same mutex removed the race

### `main-deadlock.c`
- Helgrind reported a lock order violation involving `m1` and `m2`
- One thread locked the mutexes in the opposite order from the other
- Running the program normally did not visibly hang, but the deadlock risk is still present

### `main-deadlock-global.c`
- Helgrind again reported a lock order violation
- Running the program normally exited right away
- This showed that Helgrind can detect suspicious synchronization patterns even if the bug does not visibly happen every run

### `main-signal.c`
- The program output appeared correct
- Helgrind reported a race on the shared variable `done`
- The code is inefficient because the parent busy waits instead of blocking

### `main-signal-cv.c`
- The program output was correct
- Helgrind reported 0 errors
- The mutex and condition variable version fixed the race and removed the busy waiting

---

## Conclusion

This lab showed how Helgrind can help detect concurrency-related problems such as races, deadlocks, and signaling mistakes in threaded programs. It also showed that debugging tools are helpful, but they still need to be interpreted carefully because the tool output has to be compared with what the program is actually doing.
