# File Abstractions in Linux
* In Unix-derived OSes, user level-code often interacts with important resources via the file abstraction.
    * Ex: An actual file like `/home/alice/data.txt`
    * Ex: A pipe created by `pipe()`
    * A raw storage device like `/dev/sda`
* In linux, many devices are accessible via the `/dev` directory.
    * Ex: A terminal like `/dev/tty`, which represents the terminal for the current proc.
        * The shell command `echo hello > /dev/tty` will write "hello" to the terminal.
## Virtual File System (VFS) Interface 
* In the old days, an OS could only use a single, baked-in file system
* The VFS interfaces defines an abstract, generic interface that a file system should present to the OS.
    * A specific file system has to implement the VFS methods
    * The OS only interacts with the file system through those VFS methods.
* VFS abstractions can also encapsulate "file systems" that don't involve a disk.
    * Pipes use a pure in-memory buffer

## Why use `libc` instead of directly invoking system calls?
* Portability
* Convenience
* `malloc()/new` and `free()/delete` hide the complexity of heap-related system calls like `sbrk(), mmap(), munmap()`. 
* `libc` has some caching benefits. Avoids many context switch overheads.
## Case Study: Linux's "vnode"

* `__randomize_layout` is used to make it harder for malicious code to make write /read attacks on offsets
* 