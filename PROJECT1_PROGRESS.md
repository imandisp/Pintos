# Pintos Project 1 Learning and Progress Journal

This file records what has been learned and completed for Project 1: Threads.
It should be updated after each future implementation step.

## Working Agreement

- The student types and saves changes to the Pintos source code.
- The assistant explains the problem, design, code, and tests before source code
  is changed.
- The assistant may inspect files and update this progress journal.
- Source files should not be edited by the assistant unless the student
  explicitly requests it.

## Project Requirements

Project 1 contains three main tasks:

1. Implement an Alarm Clock without busy waiting.
2. Implement priority scheduling and priority donation.
3. Implement the 4.4BSD multilevel feedback queue scheduler (MLFQS).

## 1. Alarm Clock

**Status:** Implemented and committed.

**Source file:** `src/devices/timer.c`

**Commit:** `9e5eb92` — `in thread sleeping instead of busy waiting, block the thread.`

### Problem

The original `timer_sleep()` repeatedly checked whether enough time had passed
and called `thread_yield()` while waiting. This is busy waiting: the thread did
no useful work but continued using scheduler and CPU time.

### Solution

Sleeping threads are now blocked and stored in an ordered list. Each sleeping
thread has a record containing its thread pointer and absolute wake-up tick.
The timer interrupt checks the front of this list on each tick and unblocks all
threads whose wake-up time has arrived.

The flow is:

```text
timer_sleep(duration)
    -> calculate wake-up tick
    -> insert a sleep record into the ordered sleepers list
    -> block the current thread
    -> timer interrupts continue occurring
    -> remove and unblock the thread when its wake-up tick arrives
```

### Main Changes

- Added `struct sleep_record` to store a thread and its wake-up tick.
- Added the `sleepers` list for all blocked sleeping threads.
- Initialized `sleepers` in `timer_init()`.
- Replaced the busy-wait loop in `timer_sleep()` with ordered insertion and
  `thread_block()`.
- Extended `timer_interrupt()` to wake expired sleepers with
  `thread_unblock()`.
- Added `wake_tick_less()` to order sleepers by wake-up time.
- Disabled interrupts briefly while inserting and blocking, preventing the
  timer interrupt from observing a partially completed operation.
- Made zero and negative sleep durations return immediately.

### Concepts Learned

- A blocked thread does not consume CPU time.
- Timer ticks are the time unit used by Pintos.
- Interrupt handlers cannot sleep, but they can unblock threads.
- Shared data used by both thread code and an interrupt handler must be
  protected by briefly disabling interrupts.
- Pintos lists store embedded `struct list_elem` values.
- `list_entry()` converts a list element back to its containing structure.
- An ordered list allows the interrupt to check only the earliest wake-up
  record instead of scanning every sleeping thread.

### Verification

Build from the threads directory:

```bash
cd /home/imandi/pintos/src/threads
make
```

Run the Alarm Clock tests with QEMU because Bochs is unavailable in this
environment:

```bash
cd /home/imandi/pintos/src/threads/build
make check SIMULATOR=--qemu \
  TESTS='tests/threads/alarm-single tests/threads/alarm-multiple tests/threads/alarm-simultaneous tests/threads/alarm-zero tests/threads/alarm-negative'
```

Tests relevant to this stage:

- `alarm-single`
- `alarm-multiple`
- `alarm-simultaneous`
- `alarm-zero`
- `alarm-negative`
- `alarm-priority` will also depend on the upcoming priority scheduler.

## 2. Priority Scheduling and Donation

**Status:** Basic ready-queue scheduling completed; synchronization and
donation remain.

### 2.1 Basic Priority Scheduling

**Status:** Implemented and verified, but not yet committed.

**Source file:** `src/threads/thread.c`

#### Problem

The original scheduler added ready threads to the back of `ready_list`, so it
scheduled them by arrival order instead of priority. It also allowed a running
lower-priority thread to continue after a higher-priority thread was created.

#### Solution

The ready list is now kept in descending priority order. The scheduler already
removes the front element, so this makes it select the highest-priority ready
thread. A running thread also yields immediately after creating a
higher-priority thread or lowering its own priority below another ready thread.

#### Main Changes

- Added `priority_more()` to compare two threads by priority.
- Changed `thread_unblock()` to insert ready threads with
  `list_insert_ordered()`.
- Changed `thread_yield()` to return a yielding thread to its correct ordered
  position.
- Changed `thread_create()` to yield immediately when the new thread has a
  higher priority.
- Changed `thread_set_priority()` to yield when the current thread is no longer
  the highest-priority runnable thread.
- Preserved FIFO behavior among threads with equal priorities by using a strict
  `>` comparison.

#### Concepts Learned

- A priority scheduler should always choose the highest-priority ready thread.
- An ordered ready list makes the highest-priority thread available at the
  front.
- Preemption means replacing the running thread when a more important thread
  becomes runnable.
- Ready-list access and related scheduling decisions must be protected against
  timer interrupts.
- Equal-priority threads should run in round-robin/FIFO order.

#### Verification

A clean rebuild succeeded. The following tests passed using QEMU:

- `priority-change`
- `priority-preempt`

The existing warnings came from supplied Pintos library/device files, not from
the new `thread.c` code.

### Remaining Priority Work

Planned order:

1. Respect priorities in semaphores and condition variables.
2. Track base and effective priorities.
3. Implement multiple and nested lock priority donation.
4. Remove donations when their associated lock is released.

## 3. Advanced Scheduler (MLFQS)

**Status:** Not started.

This stage will be started only after the basic priority scheduler is working.

## Next Activity

Implement and test priority-aware semaphores and condition variables so the
highest-priority waiter wakes first and can immediately preempt a lower-priority
running thread.
