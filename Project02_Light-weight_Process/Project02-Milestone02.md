# Objective of milestone02
+ To make basic LWP operations(system calls)

# LWP
+ Light Weight Process
+ A process that shares resources such as address space with other processes in the same LWP group
+ LWPs in the same LWP group also share page table
  + Therefore, newly created LWP doesn't have to allocate new page table and can just use the existing process's page table
+ LWPs has its own stack even if they are in the same LWP group
  + Therefore, newly created LWP has to allocate its own stack, and of course it has its own `esp`

# `struct proc`
+ There are some added member variables in `struct proc`
+ `int lwpid`: id of lwp, 0 for normal process
+ `int lwp_master_pid`: pid of master of LWP group, 0 for normal process
+ `struct proc* master`: master process of LWP group
+ `void* retval`: return value of LWP when executes `thread_exit`
+ `uint stack_base`: stack base of the LWP
+ `struct joined_stack_bases jsb`: store stack_base of joined LWP in the past, only master process uses it
```c
struct joined_stack_bases {
  uint addr[NPROC];
  int cnt;
};
```

# Create Description
+ `int thread_create(thread_t *thread, void *(*start_routine)(void*), void *arg)`
+ `thread`: return the thread id
+ `start_routine`: the pointer to the function to be threaded. The function has a single argument: pointer to void
+ `arg`: the pointer to an argument for the function to be threaded. To pass multiple arguments, send a pointer to a structure
+ `ret_val`: On success, thread_create returns 0. On error, it returns a non-zero value
+ It has lots of common things with xv6 `fork` and `exec` system call

# Create Implementation
```c
struct proc *curproc = myproc();
struct proc* master;
if(curproc->lwpid == 0) { // when normal process or master calls thread_create
  master = curproc;
}
else if(curproc->lwpid > 0) { // when member of LWP group calls thread_create
  master = curproc->master;
}
else {
  cprintf("lwpid cannot be negative\n");
  return -1;
}
```
+ If calling process is normal process or master process of LWP group, `master = curproc`
+ If calling process is member of LWP group, `master = curproc->master`
+ If `lwpid`of `curproc` is negative, it will be treated as an error(`return -1`)

```c
// Allocate process.
if((np = allocproc()) == 0){
  return -1;
}

np->pgdir = master->pgdir; // copy the page table of the master process
np->parent = curproc; // parent is just a calling process

np->master = master;
np->lwp_master_pid = master->pid;

np->lwpid = nextlwpid++;
*thread = np->lwpid;

if(master->jsb.cnt > 0) { // if there exists joined LWP in this group in the past, use the joined LWP's address space
  tmp_stack_base = master->jsb.addr[--(master->jsb.cnt)];
}
else if(master->jsb.cnt == 0) { // if there doesn't exist joined LWP in this LWP group in the past, just allocate new space
  tmp_stack_base = master->sz;
  master->sz += 2 * PGSIZE;
}
else {
  cprintf("joined_stack_base count cannot be less than 0\n");
  return -1;
}

if((np->sz = allocuvm(np->pgdir, tmp_stack_base, tmp_stack_base + 2 * PGSIZE)) == 0) { // allocate stack for created process by using allocuvm()
  // 2 pages are allocated: one for lwp's stack, the other for guard page
  kfree(np->kstack);
  np->kstack = 0;
  np->state = UNUSED;
  return -1;
}
clearpteu(np->pgdir, (char*)(np->sz - 2 * PGSIZE));

np->stack_base = tmp_stack_base;
```
+ Allocate basic process to `struct proc* np`
+ Assign lwpid to LWP in the same way xv6 assigns pid to process
+ Return the thread id to `thread`
+ If there exists joined LWP in this group in the past(which means master process's `jsb` has at least one data), use the joined LWP's address space
  + In this case, master->sz doesn't increase
+ If there doesn't exist joined LWP in this LWP group in the past, just allocate new space
+ According to the above two instructions, `stack_base` of newly created LWP is determined
+ Allocates stack for created process by using `allocuvm()`
  + one PGSIZE is for LWP's stack, and the other for guard page
  + if `allocuvm()` returns 0 due to some error, free the `kstack` which was allocated in `allocproc`
+ `clearpteu` is used to create an inaccessible page beneath the stack

```c
*np->tf = *master->tf;

// Set esp of trapframe of forked process to point the address of allocated stack
np->tf->esp = np->sz - 4;
*((uint*)(np->tf->esp)) = (uint)arg; // save arg pointer in stack
np->tf->esp -= 4;
*((uint*)(np->tf->esp)) = 0xffffffff; // fake return PC

// Set eip of trapframe of forked process to point the address of start_routine
np->tf->eip = (uint)start_routine;
```
+ First, copy the `trapframe` of master
+ `esp` of created LWP should be changed to point its own stack
+ Push argument and fake return PC value to stack
+ `eip` of created LWP should be changed to point `start_routine`

```c
for(i = 0; i < NOFILE; i++)
  if(master->ofile[i])
    np->ofile[i] = filedup(master->ofile[i]);

np->cwd = idup(master->cwd);

safestrcpy(np->name, master->name, sizeof(master->name));

acquire(&ptable.lock);

np->state = RUNNABLE;

release(&ptable.lock);

return 0;
```
+ Above code is the same as what `fork` does

# Exit Description
+ `void thread_exit(void *retval)`
+ `retval`: return value of the thread
+ It has lots of common things with xv6 `exit` system call

# Exit Implementation
```c
if(curproc->lwp_master_pid <= 0) {
  cprintf("only LWP member can call thread_exit()\n");
  return;
}
```
+ If process which is not an LWP member calls thread_exit, exception handling is needed

```c
if(curproc == initproc)
  panic("init exiting");

// Close all open files.
for(fd = 0; fd < NOFILE; fd++){
  if(curproc->ofile[fd]){
    fileclose(curproc->ofile[fd]);
    curproc->ofile[fd] = 0;
  }
}

begin_op();
iput(curproc->cwd);
end_op();
curproc->cwd = 0;
acquire(&ptable.lock);
```
+ Above code is the same as what `exit` does

```c
curproc->retval = retval; // store return value in calling LWP's retval

// Master might be sleeping in thread_join().
wakeup1(curproc->master);
```
+ Store return value in calling LWP's `retval`
+ We should wakeup sleeping LWP master(in xv6 `exit`system call, it wakeups parent)

```c
// Jump into the scheduler, never to return.
curproc->state = ZOMBIE;
sched();
panic("zombie exit");
```
+ Above code is the same as what `exit` does

# Join Description
+ `int thread_join(thread_t thread, void **retval)`
+ `thread`: the thread id allocated on thread_create
+ `retval`: the pointer for a return value
+ `ret_val`: On success, this function returns 0. On error, it returns a non-zero value
+ It has lots of common things with xv6 `wait` system call

# Join Implementation
```c
if(curproc->lwpid != 0) { // only master of LWP can call thread_join()
  cprintf("LWP member cannot call thread_join()\n");
  return -1;
}
```
+ If an LWP member calls `thread_join`, exception handling is needed

```c
acquire(&ptable.lock);
for(;;){
  // By looping over ptable,
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->lwpid != thread) { // find an LWP with an lwpid that matches the given argument thread
      continue;
    }
    if(p->master != curproc) { // check if calling process is master of picked LWP
      continue;
    }

    if(p->state == ZOMBIE){
      // clean up page table allocated memories, and stack of the LWP
      kfree(p->kstack);
      p->kstack = 0;
      *retval = p->retval;
      p->sz = deallocuvm(p->pgdir, p->stack_base + 2 * PGSIZE, p->stack_base); // instead of freevm in wait()
      curproc->jsb.addr[curproc->jsb.cnt++] = p->stack_base;
      p->pid = 0;
      p->parent = 0;
      p->name[0] = 0;
      p->killed = 0;
      p->state = UNUSED;

      p->lwpid = 0;
      p->lwp_master_pid = 0;
      p->master = 0;
      p->retval = 0;
      p->stack_base = 0;
      release(&ptable.lock);
      return 0;
    }
  }
```
+ Find an LWP with an `lwpid` that matches the given argument `thread` by looping over `ptable`
+ Also, found LWP's master should be the calling process
+ And also, we only carry out cleaning up when found LWP's `state` is `ZOMBIE`
+ Before cleaning up, store the stack_base of process in master process's `jsb` to reuse it in the future
+ Clean up allocated memories
+ Especially, allocated stack for picked LWP should be deallocated by `deallocuvm`(in xv6 `wait` system call, it deallocates all `pgdir` by using `freevm`)
+ We don't need to decrease master's sz because it will be reused later
+ for(;;) loop lasts to the next code

```c
// No point waiting if we don't have any LWP member of given condition(about lwpid, master, and state).
  if(curproc->killed){
    release(&ptable.lock);
    return -1;
  }
  // Wait for LWP member to exit.  (See wakeup1 call in proc_exit.)
  sleep(curproc, &ptable.lock);  //DOC: wait-sleep
```
+ If there is no such LWP, return immediately
+ And then the calling process(`curproc`) waits for LWP member to exit