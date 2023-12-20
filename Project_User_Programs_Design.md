# Project User Programs Design
Group 24

| Name         | Autograder Login | Email                     |
| ------------ | ---------------- | ------------------------- |
| Anusha Muley | student143       | anusha.muley@berkeley.edu |
| Cindy Yang   | student258       | cindycy@berkeley.edu      |
| Qi Li Yang   | student85        | qiyang2@berkeley.edu      |
| Sydney Tsai  | student134       | sydneytsai@berkeley.edu   |

----------
# Argument Passing

process_execute takes in a string with all the flags that were passed in. split on the whitespace and return argc (count of args) and argv (array of strings flags)

## Data Structures and Functions
    char *strtok_r(char *str, const char *delim, char **saveptr)

The strtok_r function will be used to separate a string based on a delimiter (in our case “ “)


    char *argv

Linked list of arguments passed into the process_execute function


    int argc

Counter variable that tells us the number of arguments passed into the process_execute function.

## Algorithms

First, we will split the string passed into process_execute using the strtok() function. This function separates each argument, allowing us to separate each argument and place them into our argv pointer and then push onto the stack along with their addresses, null pointer, and stack alignment (as stated in the program startup spec). Then, we push the address of the start of our argv pointer (the address of our argv list), as well as argc (which will be a counter variable that increments by 1 as we parse through each argument from strtok_r, keeping track of the number of arguments that are passed into argv) onto the stack. Finally, we push the return address to finalize setting up the user stack for argument passing.

## Synchronization

Using the strtok_r function allows us to parse in a thread safe fashion because it is a reentrant function that allows us to keep track of the location in the string we have parsed to.

## Rationale

Creating argv as a linked list allows us to modify the length of argv as we parse through each argument.

----------
# Process Control Syscalls
## Data Structures and Functions


    //in src/userprog/process.h
    struct process_stuff{
      struct pid_t pid // uuid of the process
      struct tid_t tid // thread id of a child thread of the process
      struct list shared_data; //linked list of shared_data of children 
      struct list of child_procs; // to keep track of what child threads are there
    // is_loaded boolean with semaphore 
    }
    //struct for both parent and child
    struct shared_data {
      struct pid_t pid //uuid of the child process
      sem_t semaphore; 
      pthread_mutex_t lock; // lock for the reference count
      int ref_cnt;
      int exit;
    }
    // in thread.h
    #ifdef USERPROG
      /* Owned by process.c. */
      struct process* pcb; /* Process control block if this thread is a userprog */
      struct process_stuff proc_info;
      struct list children; //take this out
    


| Syscall                           | Functions/DS                                                                                               |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| practice                          | N/A<br>return i + 1                                                                                        |
| halt                              | `shutdown_power_off`() in `devices/shutdown.h`                                                             |
| exit                              | process_exit() in process.c                                                                                |
| pid_t exec (const char *cmd_line) | pid_t process_execute(const char* file_name);<br>`int wait (pid_t pid)`<br>child: `void exit (int status)` |
| wait                              |                                                                                                            |



## Algorithms

Need to error check the syscall, make sure it’s not NULL and not in kernel memory

The exit syscall returns the status parameter to kernel. It calls process_exit(), then prints 
“%s: exit(%d)\n” s= process name, d = exit code. Also close all of the files before exiting and deallocate. update the wait status and decrement each of the reference counts from parent or child, check if ref_cnt is 0 

the first shared_data struct in the child list is itself, set its exit to status , and decrement its ref count

The exec syscall: 
Initialize ref_cnt to 2. Call process_execute() on the given command line arguments to create a child process. If process_execute returns TID_ERROR, return -1.  If the child process successfully loads, and cannot run then it will set the exit member of its shared_data struct to 1 and up its semaphore. If it runs, set the exit member to 0 and up semaphore. Lock the mutex and decrement the ref_cnt member before calling exit syscall. For calling process, loop through the list of child processes and call the wait syscall on each one. Calling process will obtain the child process’s exit status from the wait call. If exit member is 0, return child pid. Otherwise, return -1. Calling process acquires lock and decrements ref_cnt before returning. 


The wait syscall: 
Checking if pid is a direct child: loop through the linked list of children. Check if it exists. 
If pid is direct child, down the semaphore and return the child’s exit status. If pid is not in the linked list of child pids, return -1. 
we will be calling process_wait
decrement the reference count here
Return -1 if the child is not a direct child. 

For shared_data struct:

- Initialize lock in start_process
- Initialize exit in start_process to -1

Loop through linked list of children (shared_data) and check if direct child pid exists
decrement child refcount 
If ref_cnt == 0 , deallocate mem
If exit at default val -1, process did not call exit , return -1

things we are missing:

- If exit at default val -1, process did not call exit, return -1  
- [exit]Decrements reference count of the shared data with the parent, and destroys shared data if 0. **(check)**
- [exit]Loop through and Decrements reference count of the shared data with each child, and destroys shared data if 0.

Simplified (j writing shit)

- get the children of the current thread = child
- initially set int exit = -1
- if !child then
    - 
## Synchronization

The wait syscall essentially allows synchronization of threads because it waits for the child thread to finish before returning it’s exit status to the calling process. It utilizes a semaphore in protecting the state of the child’s exit status as well. 

For exec syscall, utilize a semaphore with initial value 0. Child process will up semaphore before it returns. The parent process will have to decrement semaphore before being able to return to ensure it obtains an exit status from the child. The parent will also wait for all children to finish by using an integer ref_cnt that is protected by a lock. 

## Rationale

By using semaphores for scheduling, we can ensure that waiting threads will not be able to run until the semaphore from the main thread signals that the process is done. 


----------
# File Operation Syscalls
## Data Structures and Functions
    //typedef for the file descriptor table, which contains file descriptor structs
    typedef struct fdt {
      struct list fd_ll; // linked pintos list of all the fds
      int size;
    } fdt;
    
    //file descriptor struct that contains the integer name and position of current pointer
    struct fd {
      int index;
      int position;
      FILE* fileptr: //pretty sure we need this to refer to file given fd
      struct list_elem* list_elem;
    };

Functions:

    From src/filesys/filesys.h:
    
    bool filesys_create(const char* name, off_t initial_size);
    struct file* filesys_open(const char* name);
    bool filesys_remove(const char* name);
    
    // from src/filesys/file.c
    file_deny_write() // sets the file struct deny_write boolean to true
    file_allow_write() // releases lock
    
    void filesys_init(bool format);
    void filesys_done(void);
    
![](https://paper-attachments.dropboxusercontent.com/s_74D1B6BE2CD6F4DA9697A26BC367EE12AE55002441485F589697E7E4D5C5453F_1695322412492_IMG_1A0541237DF9-1.jpeg)

## Algorithms
    bool create (const char *file, unsigned initial_size){
    // create a file with init size 
    bool filesys_create(const char* name, off_t initial_size){}
    // from filesys.c
    }

Need to validate read/write arguments

Create a basic file with the given initial size using the filesys_create(const char* name, off_t initial_size) function. We dont need to open it or add it to the fdt yet, just make a new file from scratch. 

For all of these functions, we are able to check if the integer passed into fd is in the file descriptor table by making sure that it is within the size value that we set in our fdt struct.


    bool remove (const char *file)

Remove a file from the filesystem given the name of the file. We again do not need to access the fdt just delete it using the filesys commands using the filesys_remove function.


    int open (const char *file)

The first time we need to open a file in a process, we have to keep a process specific fdt that tracks the file descriptors open and updates across threads. Therefore, if a per process fdt doesn’t exist, we need to initialize a new fdt. Then each time we open a file even if its already open, we create a new fd and add it to the fdt linked list, as well as update the size.


    int filesize(int fd)

If the file is in the fdt, use the file descriptor table and step through the linked list to get the pointer to the file descriptor. Then use the actual pointer to the file to call file_length and return it. 


    int read (int fd, void *buffer, unsigned size)

If the fd is 0 (or the STDIN_FILENO), we read using input_getc. Else, if the file is in the fdt, use the file_read method to read into the buffer given the pointer to a file from the file descriptor table. Return the bytes read.


    int write (int fd, const void *buffer, unsigned size) {
    if fd not in fdt return -1
    use the fd to get the pointer to the file
    call file_deny_write()
    if fd is 1, write all of buffer to putbuf function 
    call file_write(struct file* file, const void* buffer, off_t size)
    call file_allow_write()
    return bytes written (from file_write)
    }

If fd is 1, we use the putbuf function. Else, if we want to write to a file, we first validate the file descriptor and get the pointer to the file (once again, only if fd exists in fdt). Then to make sure another thread/process cannot interrupt we call file_deny_write on the pointer to the file. We write all the buffer to the file before re-enabling writes to the file and returning the number of bytes written (if the buffer is larger than a few hundred bytes, then we can use a for loop to write the buffer in chunks). 


    void seek (int fd, unsigned position)

If the integer fd passed into the seek syscall is not within the range of the size of our fdt, we end the function and do nothing. Else, the function should call the file_seek function, passing in the file associated with the fd and offsetting to the specified position.


    unsigned tell(int fd)

For the tell syscall, the function should check to see if the file descriptor is in the file descriptor table (by checking if the int is within the range of the size of our fdt). If not, the function would return -1. Else, the function calls the file_tell function, passing in the file pointer associated with the fd file pointer. The return position from file_tell is returned in this function as well to indicate the position of the next byte to be written/read to.


    void close (int fd)

For the close syscall, the function should check to see if the file descriptor is in the file descriptor table. If not, the function would return -1. Else, we can call the file_close function to close the specified file associated with fd, passing in the file pointer we have stored in our struct for that specific fd integer.

- will also remove the file index, but there could still be an open file if another descriptor points to it
## Synchronization

There should be one file descriptor table per process that is shared across the threads for that process. There is a file descriptor object for each file each time it is opened and those are different initializations across different calls to open. When another thread/process is attempting to write to the file, they must first acquire the lock for that file to ensure that there is proper synchronization.

For the tell syscall, we will need to lock the file descriptor in case another thread read or writes the file while the current thread is calling the tell syscall. If this happens, the current position of the file could be changed, forcing the tell syscall to return the wrong position. Similarly for the file_write call we call file_deny_write while we are writing to a file to ensure that we are not interrupted or writing at the same time. 


## Rationale

The file descriptor table per process allows threads within the process to gain access to file descriptors initialized in different threads. We used a linked list struct within the fdt struct in order for the list of fd to be appendable. We also included a size variable to let us know how many fd are in the fdt so when we are checking for fd in the fdt, we can easily get the number of total fd and reference the needed fd number with the total size of the fdt (This prevents us from having to traverse the entire linked list to get the size). The fd struct allows us to keep track of each individual fd through the index int, the location of the pointer, as well as the actual FILE pointer.


----------
# Floating Point Operations
## Data Structures and Functions
    copy the existing struct interrupt frame, which includes the 108-byte FPU save state
    
    //typedef for the each variable pushed onto the fpu
    struct regvals{
      float variable; 
    }
## Algorithms

Whenever we create a new thread, we need to initialise a new floating point operation. We will do this by calling fninit. However, if we already have an fpu initialized and need to create a clean fpu for a new thread or to context switch, we will need to save the current fpu and be able to restore it. Calling fsave (address) will save the current fpu state into the address passed into the function. Similarly, we will use frstore in order to restore the process if we switch back.

- initialize in init.c or start.s
- save parent FPU values into registers when creating new child
    - create an array for saving


    double compute_e (int n)

For this syscall, we can use sys_sum_to_e(n), which will return the Taylor summation of the first n+1 terms. This will be stored into the eax register.

- make sure n is not a negative number (check before sum_to_e)
## Synchronization

The FPU must be clean for each new thread/process.

- to do this, we save the current FPU registers into a temporary location, initalize a clean state of FPU registers, save these FPU registers into the desired destination, and restore the registers from the temporary location
## Rationale

The regvals struct will allow us to create new variables to be pushed onto the stack for floating point operations

----------
# Concept check
1. In src/tests/userprog/sc-bad-sp.c there is a system call with the %esp set to a bad address. In line 16 of the program, there is assembly code to change the stack pointer toa an address outside of the c program. We are moving the esp down by 64MB which is outside of the page sizes we can set for a program. This is an invalid stack pointer to test that a syscall can return an error. 
    
2. In src/tests/userprog/sc-bad-arg.c line 12 first sets the stack pointer to the same address as the SYS_EXIT system call that is being invoked. This causes the system call to be pushed into inaccessible space on the stack. The test checks this bad stack pointer placement and returns an error.
    
3. There aren't any tests that I could find for file syscall methods filesize() and tell(). For the filsize, we should have checks for files that dont exist, size 0, and edge cases. For file_tell(), we can similarly test for files that don’t exist, and files in which the position pointer is at the end of the file. 

