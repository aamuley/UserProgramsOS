# Project User Programs Report
Group 24

| Name         | Autograder Login | Email                     |
| ------------ | ---------------- | ------------------------- |
| Anusha Muley | student143       | anusha.muley@berkeley.edu |
| Cindy Yang   | student258       | cindycy@berkeley.edu      |
| Qi Li Yang   | student85        | qiyang2@berkeley.edu      |
| Sydney Tsai  | student134       | sydneytsai@berkeley.edu   |

----------
# Changes
## Argument Passing

We changed our algorithm for argument passing a bit by first calculating the amount of arguments we have by incrementing the argc variable. Then, we started adding in the arguments into our argv array.

While we were pushing arguments onto the stack, we also used a char* ptrs2args array that contained all the addresses of each argument we were pushing onto the stack so we can also push these addresses onto the stack after.

Then, we stack aligned to 16 bytes (which we did not add in our design doc). The argument addresses we stored in ptrs2args were pushed onto the stack along with out argc.

Lastly, we also had to pass in the correct process name by copying only the first word preceding the first space in the file_name into t→pcb→process_name.

Overall, we changed our design by adding to our algorithm section of the argument passing task and calculating argc before setting up argv (rather than our idea of a linked list).

## Process Control Syscalls

We got rid of the process_stuff data structure because we noticed a lot of that information was repetitive with what was already defined in the process struct. Instead of a separate data structure we merged those elements into the predefined process struct. Additionally we added a pointer to the executable so that we could deny writes. In the shared data struct we added a semaphore and a boolean to keep track of whether wait had been called. 


As for the shared data structure, we also added a parent pointer in process struct for the child process to point to the shared data struct with its parent instead of adding it to its own child list. This way, we could ensure that the child process can always access the shared parent file but not the other way around. 

We also validated all of the arguments to wait and exec to make sure that they were located in valid user memory.

In exec, we removed the logic that should’ve been in wait. In the exit sycall, we added that we had to set the exit status in the parent shared data struct and loop through all the children to check and decrement their reference count. In the wait syscall we added logic to check if the process had already called wait on the child.  We also added the check for the reference count and freed the child shared data struct if reference count got to 0. We set the default value of the exit status as -1 so that we could check if the child process ever called exit. We return -1 if not. 

Additionally we added argument and pointer validation for the exec syscall to check if the pointers are invalid, illegal or null. 


## File Operation Syscalls

Added a struct list_elem* list_elem to the struct fd to be able to use a pintos linked list for the fdt. We also realized the struct file keeps track about the position within a file so we removed that from the fd struct. We made a helper method to get the file pointer from the file descriptor number by stepping through the list

We attempted to use file_allow and file_deny write however we ran into issues when trying to instantiate it and decided to use a global file lock that protects executable files when we call exec. We then included a file deny write in the load function to make sure we dont modify executables and then release that later. Having a global file lock to call the file allow/deny functions makes sure there isnt a race to modify files. 

In the close function we added a call to free the file descriptor and remove it from the fdt. 
In process_exit we also included calls to free any file descriptors still in the fdt along with the fdt. 

Additionally we made methods in syscall. c to validate pointers and strings that we are using to make sure they are in user memory and they have enough space allocated using the is_user_vaddr() and pagedir_get_page() functions. We also have checks in process.c to check that the memory we use is below PHYS_BASE. 


## Floating Point Operations

In order to set up the floating point ops correctly, we added fpu state variables in the switch_threads_frame and intr_frame structs so that we could save the states in these variables (we had to make sure to modify the offsets in intr-stubs.h and switch.h). We also had to add the pushing and popping of these registers in the assembly code, since we added variables in the structs. Moreover, I needed temporary fpu save registers in start_process and create_thread for when I’m initializing a new fpu (this helps save the current fpu state when I am creating a clean one for the child). 

----------
# Reflection

Sydney: I worked on the argument passing, fpu task, and some file syscall stuff along with Anusha. Overall, there was a big learning curve during this project because there were so many parts that depended on other parts that the group had to learn how to adapt this new type of group work and accomplish tasks cohesively. I believe that near the end of the project, we got a good hang of utilizing git (branching, merging, pushing, etc.) to help us parallelize our tasks. However, we definitely could have used this knowledge earlier on (this will be kept in mind for future projects).
Anusha: I mostly worked on the file-sys suite and I implemented most of the syscalls for that and I also helped with the arg passing and stack aligning portions. For filesys i also worked on the data structures and wrote the helper methods to validate pointers, strings, and get file pointers in our syscall handler. I was also sick for the last 5 days we were working on this project and I think my groupmates had to do a lot of the debugging/writing tests/office hours while I was recovering. I think we didnt realize how dependent a lot of the parts were on each other and that the order mattered as much as it did but thats good to learn for future projects. Pair programming and maybe having more people collaborating and debugging would be good for future projects. 
Qi: I worked on the exec, wait and exit syscalls. I also implemented the data structures and necessary initialization, declaration and logic for the process syscalls code procedure. I helped with stack alignment and argument passing. I worked on debugging throughout our coding process. I think we started off rough because we were not using good practices in writing and testing our code (we would write a huge chunk of code before testing). It was also difficult to plan out which parts each of us would work on whether separately or together and keep track of who was doing what. However paired programming and live share worked well for us especially being able to split up terminals and test separate parts while making sure the different sections of code were synched with each other. 
Cindy: I worked on the practice, halt, and helped with the exec, and wait syscalls. I also helped with stack alignment and argument passing when the tests for those areas weren’t passing. I also worked on debugging exec, wait, and creating structures for the shared data throughout the coding process. It was difficult for us to start because we had separated a lot of the tasks. This meant that we couldn’t really test our code until we merged everything, causing a lot of bugs in the process. However, we found that live sharing our code worked really well so that we could work at the same time. 



----------
# Testing

Open STDOUT: This test checks whether our code errors when we try to open the STDOUT file. Since the STDOUT file is a default file already open with each process, we should not be able to open it again. As a result, our code correctly returns a -1 exit code when we try to open the STDOUT file (The first error code 0 is when the process first initializes and opens its STDOUT). If the kernel were to contain a corrupted list of file descriptors that point to corrupted file descriptor tables for a specific process, the output of the test would be different because it might’ve mistaken a regular file as the STDOUT or the STDOUT as a regular file. Moreover, if the kernel were to not be able to isolate processes correctly, the opening of files could be corrupted and we might open another processes’ file or have incorrectly indexed file descriptors.

![open-stdout.result](https://paper-attachments.dropboxusercontent.com/s_B61E30D34F79D96B024BABDE0E5996E16481CDF5D949B2934F0D579AABDA451D_1697177282190_Screenshot+2023-10-12+at+11.07.55PM.png)

![open-stdout.output](https://paper-attachments.dropboxusercontent.com/s_B61E30D34F79D96B024BABDE0E5996E16481CDF5D949B2934F0D579AABDA451D_1697177253017_Screenshot+2023-10-12+at+11.07.26PM.png)


Open STDIN: This test checks whether our code opens STDIN correctly. Since the STDIN in file should not be able to open without error, this ensures that our code correctly evaluates the STDIN file. Our test calls open() and passes in 0. The test checks to see whether the process begins, ends, and exits with code 0 when the process first starts. Then, when we open the STDIN file, we should expect to see exit code -1. If the kernel accesses the file descriptor table incorrectly, or the file descriptor table was not initialized correctly, then the STDIN file might not be in the correct arguments and would not be able to be accessed well. In addition, if the kernel tries to open uncreated files, it would also the result of our test because it could be accessing incorrect processes. Writing Pintos tests were a little bit unintuitive, since we have never written anything in a .ck file. However, we were able to reference the other tests. In addition, it was difficult to grasp how to add tests onto Pintos.

![open-stdin.output](https://paper-attachments.dropboxusercontent.com/s_B61E30D34F79D96B024BABDE0E5996E16481CDF5D949B2934F0D579AABDA451D_1697177323688_Screenshot+2023-10-12+at+11.08.40PM.png)



![open-stdin.result](https://paper-attachments.dropboxusercontent.com/s_B61E30D34F79D96B024BABDE0E5996E16481CDF5D949B2934F0D579AABDA451D_1697177366786_Screenshot+2023-10-12+at+11.09.23PM.png)


