# Section 2: Microkernels
## Reading 1: The Nucleus of a Multiprogramming system
**Process Communication**: The system nucleus administers a common pool of *message buffers* and a *message queue* for each process. The primitives available for communication between internal processes are:
* Send message: copies a message into first available buffer and delivers it to the queue of the receiver
* Wait message: Delays the requesting process until a message arrives in its queue 
* Send answer: Copies an answer into a buffer in which a message has been received and delivers it in the queue of the original sender
* Wait answer: Delays the requesting process until an answer arrives in a given buffer

In this approach, processes are in general not aware of each other, so they can appear/disappear any time. However, they need to agree on a common buffer to be able to communicate. But this enables them to send/receive more than one message at a time.

The method described above is for communication between internal processes. We also need to consider the communication between internal and external processes.

**External Processes**: The output of a document identified by a unique process name. External processes are created on request rfom internal processes. 
*Creation* is simply the assignment of a name to a particular peripheral device.


**Internal Processes**: The concepts of program loading and swapping are not part of the nucleus. As long as child processes remain within the area of their parent processes, the system does not care.

### Benefits of the microkernel approach
1. New operating systems can be implemented as other programs without modification of the system nucleus.  
2. Operating systems can be replaced dynamically, thus enabling an installation to switch among various modes of operation
3. Standard programs and user programs can be executed under different operating systems without modification.

# Section 2 Notes
* In monolithic kernels, all of the kernel is a single executable that runs in kernel mode (%cs 0).
* In microkernels, some kernel functionality is split in multiple processes 
* In microkernels, parents split their own resources for their children.
* In microkernels, we can dynamically replace parts of the operating systems.
* We can reason more easily about the microkernel correctness.
* Difference between kernel extensions / microkernels. Since the kernel is huge, there might be bugs in the parts. Make this code unpriviledged, so that bugs affect the whole system less.
* Ubuntu bootloader bug.
* MacOS Hybrid kernel