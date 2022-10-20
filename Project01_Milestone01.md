# Objective
+ Improve an original xv6 scheduler with MLFQ and Stride

# MLFQ
+ There is a 3-level feedback queue with level 0, 1, 2
  + In order to know where the process belongs to, add a member variable ```priority``` to the struct proc

+ When a process is newly created, it initially enters the highest priority queue of MLFQ.

+ A process on a higher priority queue is chosen to run
  + If Priority(A) > Priority(B), A runs and B doesn't

+ Each level of queue applies Round Robin policy with different time quantum
  + If Priority(A) == Priority(B), A & B run in RR
  + The highest priority queue: 1 tick
  + Middle priority queue: 2 ticks
  + The lowest priority queue: 4 ticks
  + In order to know how many quantums the process uses, add a member variable ```quantum``` and ```tick``` to the struct proc

+ Once a process uses up its time allotment at a given level(regardless of how many times it has given up the CPU), it moves down on queue


+ Each queue has a different time allotment
  + The highest priority queue: 5 ticks
  + Middle priority queue: 10 ticks
  + In order to know how many ticks the process spent on the queue, add a member variable ```allotment``` and ```spent``` to the struct proc

+ Priority Boost
  + Starvation
    + If there are too many processes in the highest priority queue, processes in lower level queues cannot be run
  + To prevent starvation, priority boosting needs to be performed periodically
  + After some time period S, move all the processes in the queues to the topmost queue
  + Frequency of priority boosting: 100 ticks of MLFQ scheduling

+ MLFQ should always occupy at least 20% of the CPU share

# Combine the stride scheduling algorithm with MLFQ

+ Stride scheduling
  + A scheduling mechanism that has been used to guarantee each process obtain a certain percentage of CPU time

+ When a process is newly created, it initially enters the MLFQ.

+ If a process in MLFQ wants to request a portion of CPU, the process invokes ```set_cpu_share``` system call which guarantees the calling process to be allocated that much of a CPU time

+ The total sum of CPU share requested from the processes in the stride queue can not exceed 80% of the total CPU time

# Required system calls
+ ```yield(void)```
  + yield the cpu to the next process
  + return 0

+ ```getlev(void)```
  + get the level of the current process in the MLFQ
  + return one of the levels of MLFQ(0, 1, 2), otherwise a negative number

+ ```set_cpu_share(int)```
  + inquires to obtain a cpu share
  + return 0 if successful, otherwise a negative number
  + How set_cpu_share(int) works
    + First, assign a percentage of tickets to the calling process
      + The total amount of tickets is 100
    + Second, calculate stride by the number of tickets allocated to the process
      + (stride) = 100 / (the number of allocated tickets)
      + In order to store the stride of each process, add a member variable ```stride``` to the struct proc
    + Third, execute the process with the lowest pass value
      + After executing, (pass) += (stride)
      + In order to store the pass value of each process, add a member variable ```pass``` to the struct proc
      + When a process becomes the stride process for the first time, set the pass value to the smallest pass value among different stride processes
        + If a new process enters with pass value 0, it will monopolize the CPU

# A logic to guarantee at least 20% CPU time to MLFQ
+ Assume that MLFQ is also a process and assign tickets(also applied to stride and pass value)
+ When set_cpu_share() invoked, assign tickets as requested by the calling process and then assign MLFQ the remaining tiekcts
+ If the percentage of tickets allocated to MLFQ is less than 20%, exception handling is needed
