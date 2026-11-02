# Process States and Schedulers

Oct 23, 2026 · @Angad

## Overview

A process is always in exactly one state, and three schedulers decide when it moves: the long-term scheduler admits it, the medium-term scheduler swaps it in or out of memory, and the short-term scheduler picks which ready process gets the CPU.

Textbook models differ only in how finely they cut the states: 5 states for the basic life cycle, 7 once swapping is added, and the 9-state UNIX model, which also splits "running" by CPU mode. Linux keeps a leaner set (section 3).

The unit actually scheduled is a thread: [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html) describes the scheduler as choosing among runnable threads, and Linux represents each as a `task_struct`. This note says "process" as the textbooks do; on Linux read it as "task".

## Process state models

Each model adds states only when a real system needs them: the 5-state model is the baseline, the 7-state model adds swapping, and the UNIX 9-state model adds CPU mode.

| Model | States | Added because |
| --- | --- | --- |
| 2-state | Running, Not running | The simplest view of the dispatcher. |
| 5-state | New, Ready, Running, Blocked, Exit | A dispatcher cannot just pick the longest-waiting process, because it may be blocked. |
| 7-state | 5-state + Ready/Suspend, Blocked/Suspend | The CPU is faster than I/O, so all processes may be waiting; some are swapped to disk to free memory. |
| 9-state (UNIX) | User Running, Kernel Running, Ready to Run in Memory, Asleep in Memory, Ready to Run Swapped, Sleeping Swapped, Preempted, Created, Zombie | Splits Running by CPU mode, and models swapping and parent-collected exit. |

Source: course slides following Stallings, chapter 3 ([Stony Brook CSE 306](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap3.pdf)).

### Seven-state model

"Suspend" means swapped out to disk: a Blocked process that is swapped out becomes Blocked/Suspend, and a Ready one becomes Ready/Suspend ([Wikipedia: Process state](https://en.wikipedia.org/wiki/Process_state)). The diagram colours each arrow by who decides it.

&#91;embedded content: seven-state model · arrows coloured by scheduler\]

Short-term arrows move a process on and off the CPU; medium-term arrows move it between memory and disk; the long-term arrows create it.

| Transition | Name | Happens when | Decided by |
| --- | --- | --- | --- |
| New → Ready | Admit | The system accepts the new process into memory. | Long-term scheduler |
| New → Ready/Suspend | Admit (suspend) | The process is created but kept on disk, typically because memory is tight. | Long-term scheduler |
| Ready → Running | Dispatch | The CPU is free and one ready process is chosen. | Short-term scheduler |
| Running → Ready | Time-out | The quantum expires or a higher-priority process becomes ready. | Short-term scheduler |
| Running → Blocked | Event wait | The process requests I/O or waits for a resource. | The process |
| Blocked → Ready | Event occurs | The awaited I/O or event completes. | The event |
| Running → Exit | Release | The process finishes, aborts or is killed. | The process or the OS |
| Blocked → Blocked/Suspend | Suspend | No process is Ready, so a blocked one is swapped out to make room for another process. | Medium-term scheduler |
| Ready → Ready/Suspend | Suspend | Swapping out a blocked process would not free enough memory. | Medium-term scheduler |
| Blocked/Suspend → Ready/Suspend | Event occurs | The awaited event completes while the process is on disk. | The event |
| Ready/Suspend → Ready | Activate | No process is Ready in memory, or this one outranks those that are. | Medium-term scheduler |
| Blocked/Suspend → Blocked | Activate | The awaited event is expected soon and the process outranks those in Ready/Suspend. | Medium-term scheduler |

Transition names follow the Stallings diagram ([chapter 3 slides](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap3.pdf)); who decides each arrow follows the scheduler-to-transition figure in [chapter 9](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap9.pdf).

### Nine-state UNIX model

UNIX splits Running into two states by CPU mode and adds Created, Zombie and Preempted. Arrows are labelled with the event that causes them.

&#91;embedded content: UNIX nine-state diagram · 9 states\]

Dashed boxes are swapped out to disk. Preempted and Ready to Run in Memory are the same pool, and interrupts taken in kernel mode loop back to Kernel Running (not drawn).

| State | Meaning |
| --- | --- |
| User Running | Executing in user mode. |
| Kernel Running | Executing in kernel mode: a system call, interrupt or fault. |
| Ready to Run, in Memory | Ready and in RAM; runs when the scheduler picks it. |
| Asleep in Memory | Blocked on an event, in RAM. |
| Ready to Run, Swapped | Ready but on disk; the swapper must bring it in first. |
| Sleeping, Swapped | Blocked on an event and on disk. |
| Preempted | Returning from kernel to user mode when the kernel preempts it and switches to another process. Same pool as Ready to Run in Memory, drawn separately to stress this case. |
| Created | Newly created, not yet ready to run. |
| Zombie | Terminated, but not yet collected by its parent ([ps(1)](https://man7.org/linux/man-pages/man1/ps.1.html) calls it defunct). |

User Running and Kernel Running are one process in two CPU modes, so a system call changes state here without any scheduling decision. The nine states are listed in [UNIX System V lecture notes](https://cse.buffalo.edu/~bina/cse421/spring00/lec3/tsld026.htm) and the figure is in the chapter 3 slides above.

## How Linux maps onto these states

Linux has no separate Ready and Running states, no suspend states, and no User Running or Kernel Running states. It keeps two fields per task: `__state` for runnability and `exit_state` for exiting ([include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)).

| Textbook state | Linux state | `ps` code | Notes |
| --- | --- | --- | --- |
| Ready and Running | `TASK_RUNNING` | R | One state: running or runnable on a run queue. Whether it holds a CPU is tracked in separate fields such as `on_cpu` and `on_rq`. |
| Blocked (asleep) | `TASK_INTERRUPTIBLE` | S | Waiting for an event; signals can wake it. |
| Blocked (uninterruptible) | `TASK_UNINTERRUPTIBLE` | D | Usually waiting on I/O; signals do not wake it. `TASK_KILLABLE` is the same state plus wake-on-fatal-signal. |
| Blocked (idle kernel thread) | `TASK_IDLE` | I | Defined as uninterruptible plus `TASK_NOLOAD`; `ps` calls it an idle kernel thread. |
| Suspend (closest match) | `__TASK_STOPPED`, `__TASK_TRACED` | T, t | Stopped by a job-control signal or by a debugger. This is a stop, not a swap. |
| Exit | `EXIT_ZOMBIE`, `EXIT_DEAD` | Z, X | Held in `exit_state`. A zombie has terminated but its parent has not reaped it. |
| New | `TASK_NEW` | none | Defined in `sched.h`, but not part of `TASK_REPORT`, so `ps` never shows it. |

`ps` letters and meanings are from [ps(1)](https://man7.org/linux/man-pages/man1/ps.1.html). The kernel header also defines `TASK_PARKED`, `TASK_WAKING`, `TASK_FROZEN` and `TASK_RTLOCK_WAIT` for special cases.

The UNIX-model states that are missing have these Linux counterparts:

- **User and Kernel Running:** CPU mode is not a task state. The kernel tracks it only for time accounting, with values such as `VTIME_USER` and `VTIME_SYS`.
- **Preempted:** a preempted task simply stays `TASK_RUNNING` and goes back on the run queue.
- **Swapped states:** no task state means "swapped out". Memory reclaim is done by kernel threads such as `kswapd`, marked with the `PF_KSWAPD` flag.

## Long-, medium- and short-term schedulers

The three schedulers differ in what they decide and how often: the long-term scheduler decides whether a process enters the system, the medium-term one whether it stays in memory, and the short-term one which ready process gets the CPU.

|  | Long-term | Medium-term | Short-term |
| --- | --- | --- | --- |
| Also called | Admission or job scheduler | Swapper | CPU scheduler |
| Decides | Whether a new process is admitted, which sets the degree of multiprogramming | Which process is swapped out of memory, and when it returns | Which ready, in-memory process gets the CPU next |
| Arrows on the 7-state diagram | New → Ready, New → Ready/Suspend | Suspend and Activate | Ready → Running, Running → Ready |
| Triggered by | A process being created | Memory running short or becoming free | Clock or I/O interrupt, system call, signal |
| How often | Infrequently, seconds to minutes | In between | Frequently, in milliseconds; at least once per time slice |
| Looks at | Free memory; a good mix of I/O-bound and CPU-bound processes | Inactivity, priority, page-fault rate, memory held | Priority, time slice, response time, fairness |

Sources: [Wikipedia: Scheduling](<https://en.wikipedia.org/wiki/Scheduling_(computing)>), the Stallings [chapter 9 slides](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap9.pdf), and the [Silberschatz lecture slides](https://www.slideshare.net/slideshow/processconceptspdf/256192362) for the frequencies.

### Where each one runs

- **Short-term:** ordinary kernel code, in kernel mode, on the stack of whichever task is current. On Linux the entry point is `schedule()`, called when a task blocks or exits, or after a tick or wakeup sets the need-resched flag (`sched_tick()`, `set_tsk_need_resched()` in [sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)). The dispatcher is the part that switches context: it gets control in kernel mode after an interrupt or system call, and ends by switching to user mode ([Wikipedia](<https://en.wikipedia.org/wiki/Scheduling_(computing)>)).
- **Medium-term:** memory management, not the CPU scheduler. It moves processes between main and secondary memory, and on systems with demand paging it can even take over the long-term role by treating binaries as swapped-out processes ([Wikipedia](<https://en.wikipedia.org/wiki/Scheduling_(computing)>)). Linux task states have no swapped-out state; reclaim runs in kernel threads such as `kswapd`.
- **Long-term:** admission when a process is created. Batch systems, clusters and render farms put special job-scheduler software on top of the OS for this ([Wikipedia](<https://en.wikipedia.org/wiki/Scheduling_(computing)>)). Real-time kernels use an admission test: Linux refuses a `SCHED_DEADLINE` task whose request makes the set infeasible ([sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html)). For ordinary tasks Linux has no long-term scheduler in the textbook sense; a new task simply becomes runnable, subject to limits such as `RLIMIT_NPROC`.
- **I/O scheduler:** Stallings lists a fourth kind, which picks the pending I/O request a free device handles next ([chapter 9 slides](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap9.pdf)).

## Scheduling algorithms: what runs where and why

Nearly every named algorithm is a short-term (CPU) policy; long- and medium-term decisions use simpler admission and eviction rules.

### Short-term: the classic algorithms

| Algorithm | Preemptive | Picks next | Trade-off |
| --- | --- | --- | --- |
| First come, first served | No | Oldest in the ready queue | Simple and starvation-free, but short processes wait behind long ones (convoy effect) and CPU-bound processes are favoured. |
| Round robin | Yes, by clock | Next in a cycle; each gets a fixed time slice | Good response time. Overhead rises as the slice shrinks; a very large slice behaves like FCFS. |
| Shortest process next | No | Shortest expected run time | Short processes jump ahead. Needs run-time estimates, and long processes can starve. |
| Shortest remaining time | Yes | Least time remaining | Preemptive version of the above; needs remaining-time estimates and adds context switches. |
| Highest response ratio next | No | Highest (wait + service) ÷ service | Waiting raises a process's ratio, so long processes are not starved; needs service-time estimates. |
| Priority queues | Usually | Highest non-empty priority level | Lower levels can starve; aging (raising priority with waiting time) fixes it. |
| Feedback (multilevel) | Yes | Highest queue; a process drops one level each time it uses its slice | Needs no run-time estimates. Long jobs sink, so lower levels get longer slices to avoid starvation. Used in UNIX. |
| Earliest deadline first | Yes | Process closest to its deadline | For real-time systems; needs known deadlines. |

Sources: Stallings [chapter 9 slides](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap9.pdf) and [Wikipedia: Scheduling](<https://en.wikipedia.org/wiki/Scheduling_(computing)>).

### Short-term on Linux

Linux runs several policies side by side in strict priority order, and all scheduling is preemptive: a runnable thread with higher priority always displaces a lower one ([sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html)).

| Policy | Used for | Rule |
| --- | --- | --- |
| `SCHED_DEADLINE` | Real-time jobs with deadlines | Global earliest deadline first plus a constant-bandwidth server. Passes an admission test, then preempts every other policy. |
| `SCHED_FIFO` | Real-time | Static priority 1 to 99; runs until it blocks, yields or a higher priority arrives. No time slice. |
| `SCHED_RR` | Real-time | `SCHED_FIFO` plus a time quantum within each priority. |
| `SCHED_OTHER` | Default time-sharing | Fair sharing by dynamic priority, set from the nice value. |
| `SCHED_BATCH` | Non-interactive jobs | Like `SCHED_OTHER`, but assumed CPU-bound, so mildly disfavoured on wakeup. |
| `SCHED_IDLE` | Very low priority jobs | Runs below even nice +19. |

Which fair-share algorithm serves `SCHED_OTHER` depends on the kernel: sched(7) names CFS as the default since 2.6.23, while [Wikipedia](<https://en.wikipedia.org/wiki/Scheduling_(computing)>) records EEVDF replacing CFS in Linux 6.6.

### Long- and medium-term rules

- **Long-term:** admit while memory and limits allow, and keep a mix of I/O-bound and CPU-bound processes. All I/O-bound leaves the ready queue nearly empty; all CPU-bound leaves devices idle ([Wikipedia](<https://en.wikipedia.org/wiki/Scheduling_(computing)>)). Real-time admission tests check that total utilization, the sum of Runtime ÷ Period, is at most the number of CPUs. That is necessary but not sufficient ([sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html)).
- **Medium-term:** swap out blocked processes first, and a ready one only when nothing else frees enough memory ([lecture slides on the seven-state model](https://www.slideshare.net/slideshow/processconceptspdf/256192362)). Likely candidates are inactive, low-priority, frequently page-faulting or memory-heavy processes; they return when memory frees up or they unblock ([Wikipedia](<https://en.wikipedia.org/wiki/Scheduling_(computing)>)).

## Scheduler and dispatcher: the control flow

The scheduler and the dispatcher are two roles in one kernel code path, not two threads: the scheduler picks the next process, and the dispatcher hands it the CPU.

1. **Trigger.** A clock or I/O interrupt, a system call, a signal, or the running process blocking or exiting puts the CPU into kernel mode ([chapter 9 slides](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap9.pdf)).
2. **Scheduler (policy).** The short-term scheduler chooses the next process from the ready queue, using one of the algorithms above.
3. **Dispatcher (mechanism).** It saves the old process's context, loads the new one's, switches to user mode and jumps to where the new process left off ([Wikipedia](<https://en.wikipedia.org/wiki/Scheduling_(computing)>)). The time this takes is the dispatch latency.
4. **Run.** The new process executes in user mode until the next trigger.

This code runs inside whichever process was current: operating-system software executes in privileged mode within the context of a user process ([chapter 3 slides](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap3.pdf)). On Linux the entry point is `schedule()` ([sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)). When nothing is runnable, each CPU runs its idle task, flagged `PF_IDLE`.

## Sources

All pages below were opened and read on 23 Oct 2026. I did not open the textbooks themselves; the course slides follow their chapters.

- [Stallings-based slides, chapter 3: Process Description and Control](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap3.pdf) (Stony Brook CSE 306): 5-state, 7-state and UNIX 9-state diagrams.
- [Stallings-based slides, chapter 9: Uniprocessor Scheduling](https://www3.cs.stonybrook.edu/~kifer/Courses/cse306/lectures/chap9.pdf) (Stony Brook CSE 306): scheduler types, mapping to state transitions, algorithms.
- [Silberschatz, Galvin and Gagne lecture slides, chapter 3](https://www.slideshare.net/slideshow/processconceptspdf/256192362): schedulers, frequencies, transition conditions.
- [Wikipedia: Scheduling (computing)](<https://en.wikipedia.org/wiki/Scheduling_(computing)>): scheduler types, dispatcher, algorithm properties, Linux scheduler history.
- [Wikipedia: Process state](https://en.wikipedia.org/wiki/Process_state): swapped-out states; cites Stallings ch. 3 and Bach, *The Design of the UNIX Operating System*.
- [UNIX System V lecture slide](https://cse.buffalo.edu/~bina/cse421/spring00/lec3/tsld026.htm) (Buffalo CSE 421): the nine UNIX states.
- [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html): Linux scheduling policies and admission test (man-pages 6.19).
- [ps(1)](https://man7.org/linux/man-pages/man1/ps.1.html): process state codes.
- [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h): Linux task states (master branch, fetched 23 Oct 2026).

Caveat: sched(7) still names CFS as the default scheduler, while Wikipedia records EEVDF replacing it in Linux 6.6. Confirm against your kernel's documentation before quoting either.
