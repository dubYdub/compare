# Project 2: User Programs



## I. Group Members



- Shiyi Li 11610511@mail.sustech.edu.cn

- Mengwei Guo 11610615@mail.sustech.edu.cn

  ​

## II. Division of labor

#### 

- #### Task 1

  - **Code** : Li Shiyi
  - **Report **: Li Shiyi


- #### Task 2 
  - **Code** : Li Shiyi & Guo Mengwei
  - **Report** : Li Shiyi & Guo Mengwei


- #### Task 3
  - **Code**: Li Shiyi & Guo Mengwei
  - **Report**: Li Shiyi & Guo Mengwei



## III. Implementation



*Before we started coding, we had a brief look at pintos itself. As the picture below shown,  while running a user program, pintos will follow the steps like:*

1. PintOS will  start and do the initialization, which includes storage allocation for kernel programs.
2. PintOS will load the ELF executable file and allocate address space for the user program (activate the page directory at the same time).
3. After that, the user program can take argument and push them in the stack of it. **It will be finished in Task 1.**
4. The System call function will read the parameters from the stack by moving the pointer. After executed the executable, it will put the return value in a specific position.
5. The Main process of operating system will wait for the return value of system call, when it has received the return value, the main process will be shut down.

![1558181902970](C:\Users\29551\AppData\Local\Temp\1558181902970.png)



### Task 1: Argument Passing



- #### Requirement Analysis

  While executing an executable, it always requires arguments to control the running of the program. For example, the file name, the waiting time, and the path. The pintOS follows the style of parameter passing in C program, which is set two default parameters for the main function: **int argc** and **char* argv[].** argc generation the number of table arguments, and argv is an array of pointers passed in as strings. The placement of the parameters should be on the top of the stack of the user program stack. The stack, will expand space from high position to low position.

  ​

- #### Data structures

  #### *TODO*

  #### 

- #### Relative Function

  #### *TODO*




- #### Algorithm & implementation

  ​
   - **Get the name and path of executable**

     *TODO*

     ​

  - **The time to pass the argument**

    After the  the **load()** function has finished in function **start_process()**, we can write the arguments to the top of stack.

    ​

   - **Decode the arguments string**

     *TODO*

     ​

    - **Set the value of pointer**

       After the program has been loaded, the executable's entry pointer eip will be set,as well as the initial stack pointer esp, which will point to the initial stack position by calling function setup_stack.


#### 

- #### Synchronization

  Everything in PintOS is represented by file, so while we run a specific syscall call, the user program shouldn't be modified (The file operation system call in Task 3 can do that) by other processes to prevent the changing of results. So we add a lock named **system_file_lock** in **thread.h** to implement the synchronization while loading the file.




## Task 2: Process Control Syscalls

### 2.1 Data structures and functions

- <thread.h>
  - We add some new attributes to the `struct thread`:
    - `bool load_success` : whether its child process successfully loaded its executable.
    - `struct semaphore load_sema` : keep the thread waiting until it makes sure whether the child process is successfully loaded.
    - `int exit_status` 
    - `struct list children_list` : list of its child processes
    - `struct file * self` : its executable file, we've discussed it in **Task1**
    - `strut child_process *waiting_child`: a pointer to the child process it is currently waiting on 
  - We create a new struct called `child_process` and had it some attributes:
    - `tid` : its thread id. making it easy for its parent finding it.
    - `if_waited` : whether the child process has been waited. According to the document, a process may wait for any given child at most once. The attributes would be initialized to `false`
    - `int exit_status` : its exit status. used to pass to its parent process when it is wated.
    - `struct semaphore wait_sema` : This semaphore is designed for waiting process. It is used to avoid race condition

#### 2.1.2 Functions

The functions involved :

- <syscall.c> 
  - `void * is_valid_addr(const void *vaddr);`
  - `void pop_stack(int *esp, int *a, int offset)`

To implement the three system call functions the document ask for, we write a method `syscall_handler` to handle all the system calls in both **Task2** and **Task3** by checking the argument in the stack using a `switch-case` structure. The three system calls in this task are wraped into functions `syscall_halt()`, `syscall_exec()` and `syscall_wait()`, which calls the functions `shutdown_power_off()`, `process_execute()` and `process_wait()` in the process.c file.

### 2.2 Algorithms

#### 2.2.1 Analysis

In this task, we are asked to implement three kernel space system calls. A `switch-case` structure in `system_handler` let the system calls execute their coresponding code. The type of system calls read from the syscall argument located on the user process’s stack. However, we have to make sure  the process reads and writes in the user process's virtual address space. That is, we should check whether the address is pointed to a valid address before execute system calls. 

The invalid memory access include:

- NULL pointer
- Invalid pointers (which point to unmapped memory locations) 
- Pointers to the kernel’s virtual address space 

#### 2.2.2 Implementation

##### 2.2.2.1 Check valid address

We implement it in function:

```c
void *is_valid_addr(const void *vaddr)
{
	void *page_ptr = NULL;
	if (!is_user_vaddr(vaddr) || !(page_ptr = pagedir_get_page(thread_current()->pagedir, vaddr)))
	{
		exit_process(-1);
		return 0;
	}
	return page_ptr;
}
```

The function checks whether the address is valid. If the `vaddr` is `NULL` , or of the kernel address space, or points to invalid locations, `exit_process` is called to terminate the current thread with exit status -1. Otherwise it returns the pyhsical address.

##### 2.2.2.2 Pop stack

In order to pop the argument we want from the user process's stack, we realize the poping process in a method :

```c
void pop_stack(int *esp, int *a, int offset){
	int *tmp_esp = esp;
	*a = *((int *)is_valid_addr(tmp_esp + offset));
}
```

All pop operations on the stack need to call this function. It will verify if the stack pointer is a valid user-provided pointer, then dereference the pointer of a specific location(offset) which we will discuss later.

##### 2.2.2.3 Halt

We simply calls the `shutdown_power_off()` in <devices/shutdown.c> 

##### 2.2.2.1 Exec

We call <syscall.c>`syscall_exec(file_name)` first to check whether the file refered by `file_name`  is valid.

```c
pop_stack(f->esp, &file_name, 1);
if (!is_valid_addr(file_name))
	return -1;
```

If it is not, we return -1, else we call <process.c>`process_execute(file_name)`.  In function `process_execute` , we first split the thread name and create a child process with it. We wait util the `thread_create` return. Since the `tid` returned could be invalid, the parent should wait until it knows whether the child process's executable file is loaded successfully. We implement this by `thread->load_sema` which we will discuss later in the Synchronization part. We store the loading information in `thread->load_success`. If successfully loaded, the method returns its `tid`, otherwise -1.

##### 2.2.2.1 Wait

The syscall means that  the process should wait for a child process pid and retrieves the child’s exit status. First we should pop the syscall argument (child process's tid) from the user process stack and check if it is valid:

```c
pop_stack(f->esp, &child_tid, 1);
```

Then we call <process.c>`process_wait(child_tid)`. If the child process with `child_tid` exists in `thread_current()->children_list` , then we can go on wait. We implement the finding process in a method <thread.c> `find_child_proc(child_tid)` which find the corresponding `list_elem` with `tid=child_tid` in the current thread's `children_list`. 

Note that according to the document,  a process may wait for any given child at most once.  Therefore, before we step into waiting process, we should first check whether the child has been waited before. If not, set the child thread's `if_waited` to `true` and down the semaphore `wait_sema`. The `wait_sema` will be increased in method `process_exit` when child process exits. Finally we can remove the child process from `child_list` and return its `exit_status`.

### 2.3 Synchronization

#### 2.3.1 `filesys_lock` : Lock on file system operations

According to the document: the Pintos filesystem is not thread-safe. We must make sure that the file operation syscalls do not call multiple filesystem functions concurrently. And we are permitted to use a **global lock** on filesystem operations, to ensure thread safety. Therefore, we assert the global lock `filesys_lock` in `thread.h`

In this task, we use the lock in the following place:

- <process.c>`load` : we acquire the `filesys_lock` before we allocate the page directory, open executable file and all the operations we had on file. Then we release the lock at the end of the method before it returns. Note that we have to release the lock whether the loading process is successful or not.

  ```c
  bool load (const char *file_name, void (**eip) (void), void **esp){
      lock_acquire(&filesys_lock);
      ....
      // operations on the file
      if load fail:
      	goto done
      ....
      done:
      	lock_release(&filesys_lock);
  }
  ```

- <syscall.c> `exec_process` : The method opens the file with the `file_name` to check whether the file exists. Before we open, we should acquire the lock, and release it after. Note that we should release the lock whether the file exists or not.

#### 2.3.2 `thread->load_sema`

When a process is creating a new child process, it has to wait until it knows whether the child process's executable file is loaded successfully. Therefore, once the child thread is created, it downs the `wait_sema` in method <process.c>`process_execute` and block the parent thread:

```c
sema_down(&current_thread->load_sema);
```

Once the child process'c executable file finish loading, it upped the semaphore to wake up the blocked parent thread in method <process.c>`start_process`:

```c
sema_up(&current_thread->parent->load_sema);
```

Note that we should up the semaphore whether the executable file is loaded successfully or not.

#### 2.3.3 `child_process->wait_sema`

The semaphore is decreased when a thread start to wait in <process.c>`process_wait` one of its child process:

```c
sema_down(&child_process->wait_sema);
```

 Once the semaphore is decreased, the thread is blocked until the child process exits in method <thread.c>`thread_exit`. When the child process the process is waiting exits, it upped the semaphore `wait_sema` and wake up the blocked parent thread:

```c
sema_up(&thread_current()->parent->waiting_child->wait_sema);
```

### 2.4 Rationale

In this task, we accomplished three kernel system calls. To achieve that, we add some new attributes to the struct `thread` and designed a new structure `child_process` for child process. Semaphores are also used in this task to prevent race condition and make sure the execution order.

## Task 3: File Operation Syscalls

### Data structures and functions

#### thread.h

```c
struct thread {
    ...
    struct list opened_files;   //all the opened files
    int fd_count;
    ...
};
```

Holding the list of all opened files.

#### syscall.h

```c
struct process_file {
	struct file* ptr;
	int fd;
	struct list_elem elem;
};
```

Store file pointer and file description number, as a list_elem in the `opened_files` of the `struct thread`.

#### syscall.c

- `static void syscall_handler (struct intr_frame *)`

  Handling the file syscall, going to the specific calls by the values in the stack.

- specific file syscall functions

  Call the appropriate functions in the file system library.

  ```c
  void syscall_exit(struct intr_frame *f);
  int syscall_exec(struct intr_frame *f);
  int syscall_wait(struct intr_frame *f);
  int syscall_creat(struct intr_frame *f);
  int syscall_remove(struct intr_frame *f);
  int syscall_open(struct intr_frame *f);
  int syscall_filesize(struct intr_frame *f);
  int syscall_read(struct intr_frame *f);
  int syscall_write(struct intr_frame *f);
  void syscall_seek(struct intr_frame *f);
  int syscall_tell(struct intr_frame *f);
  void syscall_close(struct intr_frame *f);
  ```

- `void * is_valid_addr(const void *vaddr)`

  Verifying `VADDR` is a user virtual address and is located in the current thread page.

- `void pop_stack(int *esp, int *a, int offset)`

  All pop operation on the stack needs to call this function. It will verify if the stack pointer is a valid user-provided pointer, then dereference the pointer.

- `int exec_process(char *file_name)`

  Sub-function invoked by `int syscall_exec()`: split string into tokens and call `process_execute()` with tokens.

- `void exit_process(int status)`

  Sub-function invoked by `int syscall_exit()`: set current thread status by `status`, and update the status of the child(current) process in its parent process. At last, call `thread_exit()`.

- `struct process_file* search_fd(struct list* files, int fd)`

  Find file descriptor and return process file struct in the process file list, if not exist return NULL.

- `void clean_single_file(struct list* files, int fd)`

  Go through the process file list, and close specific process file, and free the space by the file descriptor number.

- `void clean_all_files(struct list* files)`

  Go through the process file list, close all process file and free the space. Do this when exit a process.

### Algorithms

The Pintos filesystem is not thread-safe. File operation syscalls cannot call multiple filesystem functions concurrently. Here we add more sophisticated synchronization to the Pintos filesystem. For this project, we use a global lock defined in `threads/thread.h` on filesystem operations, to ensure thread safety. 

When a syscall is invoked, `void syscall_handler()` handle the process. All arguments are pushed in the stack when a user program doing system operation using `lib/user/syscall.c`. So, we just need to take the parameter from the stack.

1. We pop the syscall number from the stack by the `esp`.
2. we go to the specific syscall functions by the syscall number. For each file syscalls, we need to pop more detailed arguments. If the parameter we pop out is a pointer (such as a `char *`), we also need to verify its usability.
3. Each file syscalls call the corresponding functions in the file system library in `filesys/filesys.c` after acquired global file system lock.
4. The file system lock will be released anyway at last.

### Synchronization

All of the file operations are protected by the global file system lock, which prevents doing I/O on a same fd simultaneously. 

- First, we check whether the current thread is holding the global lock `filesys_lock` . If so, we release it.	

  ```c
  if (lock_held_by_current_thread(&filesys_lock))
      lock_release(&filesys_lock);
  ```

- Then we have to close all the file the current thread opens and free all its child.

  ```c
  lock_acquire(&filesys_lock);
  //close all current_thread()->opened_files
  //free current_thread()->children_list;
  lock_release(&filesys_lock);
  ```

Also, we disable the interruption, when we go through `thread_current()->parent->children_list` or `thread_current()->opened_files`, to prevent unpredictable error or race condition in context switch. So they will not cause race conditions.

### Rationale

Actually, all the critical part of syscall operations are provided by `filesys/filesys.c`. At the same time, the document warns us avoiding modifying the `filesys/` directory. So the vital aspect is that poping and getting the data in the stack correctly, and be careful not to do I/O simultaneously.



## IV.Addition questions

#### 

- #### *A reflection on the project–what exactly did each member do? What went well, and what could be improved?*

  The division of labor has been listed in the pervious part. **TODO**

- #### *Does your code exhibit any major memory safety problems (especially regarding C strings), memory leaks, poor error handling, or race conditions?*

  #### 

- #### *Did you use consistent code style? Your code should blend in with the existing Pintos code. Check your use of indentation, your spacing, and your naming conventions.*

  #### 

- #### *Is your code simple and easy to understand?*

  #### 

- #### *If you have very complex sections of code in your solution, did you add enough comments to explain them?*

  #### 

- #### *Did you leave commented-out code in your final submission?*

  #### 

- #### *Did you copy-paste code instead of creating reusable functions?*

  No, we always try to create the reusable but not use duplicated code block in different places. For example , we create function xxxx and xxxx in thread.c to implement the file search in one thread. Because of that, we don't need to implement the same 

- #### *Are your lines of source code excessively long? (more than 100 characters)*

  No, there is no such type of line in our source code.

- #### *Did you re-implement linked list algorithms instead of using the provided list manipulation*?


  No, we don't modify the basic structure of default linked list. We add two linked list named **child_list**(child thread list of every thread) and **file_list**(the file description of one thread) in the thread.h by using the provided list manipulation.

