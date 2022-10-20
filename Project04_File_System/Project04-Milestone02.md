# Objective
+ Implement the pread, pwrite system call
+ Implement a thread-safe read and write user library

# Basic description of `pread` and `pwrite`
+ `int pwrite(int fd, void* addr, int n, int off)`
  + Reads up to `n` bytes from file descriptor `fd` at offset `off` (from the start of the file) into the buffer starting at `addr`. The file offset is not changed.
+ `int pread(int fd, void* addr, int n, int off)`
  + Writes up to `n` bytes from the buffer starting at `addr` to the file descriptor `fd` at offset `off`. The file offset is not changed.

# Implementation of `pread` and `pwrite`
```c
int
sys_pread(void)
{
   struct file *f;
   int n, off; 
   char *addr;

   if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argptr(1, &addr, n) < 0 || argint(3, &off) < 0)
     return -1;
   return filepread(f, (void*)addr, n, off);
}
```
```c
int
sys_pwrite(void)
{ 
  struct file *f;
  int n, off;
  char *addr;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argptr(1, &addr, n) < 0 || argint(3, &off) < 0)
    return -1;
  return filepwrite(f, (void*)addr, n, off); 
}
```
+ `sys_pread()` and `sys_pwrite()` in `sysfile.c`
+ `argfd()` fetch the `f`th word-sized system call argument as a file descriptor and return both the descriptor and the corresponding struct file
+ The actual implementation of `pread()` and `pwrite()` is implemented by `filepread()` and `filepwrite()` in `file.c`

```c
int
filepread(struct file *f, void *addr, int n, int off)
{
  int r;
  
  if(f->readable == 0) {
    return -1;
  }
  if(f->type == FD_PIPE) {
    return piperead(f->pipe, addr, n);
  }
  if(f->type == FD_INODE) {
    ilock(f->ip);
    r = readi(f->ip, addr, off, n);
    iunlock(f->ip);
    return r;
  }
  panic("filepread"); 
}
```
+ `int filepread(struct file *f, void *addr, int n, int off)` in `file.c`
+ The above code is the same as `fileread()` except offset addition
  + The offset of the given file pointer should not be changed
+ Use parameter `off` instead of `f->off`

```c
int
filepwrite(struct file* f, void *addr, int n, int off)
{
  int r;

  if(f->writable == 0) {
    return -1;
  }
  if(f->type == FD_PIPE) {
    return pipewrite(f->pipe, addr, n);
  }
  if(f->type == FD_INODE) {
    int max = ((MAXOPBLOCKS-1-1-2) / 2) * 512;
    int i = 0;
    while(i < n) {
      int n1 = n - i;
      if(n1 > max) {
        n1 = max;
      }

      begin_op();
      ilock(f->ip);
      r = writei(f->ip, addr + i, off, n1);
      iunlock(f->ip);
      end_op();

      if(r < 0) {
        break;
      }
      if(r != n1) {
        panic("short filepwrite");
      }
      i += r;
    }
    return i == n ? n : -1;
  }
  panic("filepwrite");
}
```
+ `int filepwrite(struct file* f, void *addr, int n, int off)` in `file.c`
+ The above code is the same as `filewrite()` except offset addition
  + The offset of the given file pointer should not be changed
+ Use parameter `off` instead of `f->off`

# Basic description of thread-safe read and write
+ `pread()` and `pwrite()` doesn't change the offset of the file pointer
+ However, it doesn't prevent a race condition that occurs when threads in the process write and read the same area of the file
+ Thread-safe read and write avoid malicious race conditions, when different threads read and write to the same file within the process
+ For this, reads and writes to overlapping regions within the process must be protected by using reader-writer locks which was implemented in Project03
+ Additionally, functions related to thread-safe are user library

# Implementation of thread-safe read and write user library
```c
typedef struct your_structure {
  rwlock_t lock;
  int fd;
} thread_safe_guard;
```
+ `thread_safe_guard` defined in `types.h`
  + `rwlock_t lock`: a rwlock_t variable
  + `int fd`: a file descriptor

```c
thread_safe_guard*
thread_safe_guard_init(int fd)
{
  static thread_safe_guard tsg;
  rwlock_init(&(tsg.lock));
  tsg.fd = fd;
  return &tsg;
}
```
+ `thread_safe_guard* thread_safe_guard_init(int fd)` in `ulib.c`
+ Declare a `thread_safe_guard` structure as a static variable
+ Initialize `lock` of `tsg`
+ Map the declared structure with `fd`
+ Return the address of `tsg`

```c
int
thread_safe_pread(thread_safe_guard* file_guard, void* addr, int n, int off)
{
  rwlock_acquire_readlock(&file_guard->lock);
  int len = pread(file_guard->fd, addr, n, off);
  rwlock_release_readlock(&file_guard->lock);
  return len;
}
```
+ `int thread_safe_pread(thread_safe_guard* file_guard, void* addr, int n, int off)` in `ulib.c`
+ Execute `pread()` with given parameters
  + This reading is a critical section
+ Surround the critical section with `rwlock_acquire_readlock` and `rwlock_release_readlock`
+ Return the size of the read data

```c
int
thread_safe_pwrite(thread_safe_guard* file_guard, void* addr, int n, int off)
{
  rwlock_acquire_writelock(&file_guard->lock);
  int len = pwrite(file_guard->fd, addr, n, off);
  rwlock_release_writelock(&file_guard->lock);
  return len;
}
```
+ `int thread_safe_pwrite(thread_safe_guard* file_guard, void* addr, int n, int off)` in `ulib.c`
+ Execute `pwrite()` with given parameters
  + This writing is a critical section
+ Surround the critical section with `rwlock_acquire_writelock` and `rwlock_release_writelock`
+ Return the size of the written data

```c
void
thread_safe_guard_destroy(thread_safe_guard* file_guard)
{
  file_guard = 0;
}
```
+ `void thread_safe_guard_destroy(thread_safe_guard* file_guard)` in `ulib.c`
+ Set `file_guard` to 0 which means `NULL`