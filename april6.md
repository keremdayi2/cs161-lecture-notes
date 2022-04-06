# Virtualization
* Hypervisor (Virtual Machine Monitor), Hardware 
* Monitor: software that governs the execution of other software
## Why is virtualization is Useful
* Security: Isolation between multiple virtual machines is useful if VMs don't trust each other
* Ex: In windows 10 enterprise, navigating to an untrusted website will open the web browser inside of a hyper VM.
    * Windows creates a new copy of the kernel and the minimum windows services that are needed to run microsoft Edge.
* **Server Consolidation**: VMs make it easy for the owner of a physical machine to rent that machine to others
* Cram a bunch of somewhat busy server apps onto a single physical machine
* Cheaper than allocating a separate physical server per app
* Helps the data center model of computation work. The code in the VMs cannot corrupt the machine
* Ex: A multi-tenant (customers) datacenter like Amazon's EC2 runs code from multiple parties on the same physical server
## How can we implement virtualization?
* Hosted interpretation
* Popek-Goldberg VMMs
* Dynamic Binary Translation
* Hardware-assisted Virtualization
### Hosted Interpretation
* Run the VMM as a regular user application atop a host OS
    * VMM maintains the software-level representation of physical hardware
    * VMM steps through the instructions in teh code of the VM, updating the virtual hardware as necessary
* Like a while loop that fetches the next instruction, do a switch on what the current instruction is:
    * Emulate the results of the instructions.
    * This was used by Bocks (x86) and Gemulator (Atari)
* **Good**: Easy to interpose on sensitive operations
    * The guest OS will want to read and write sensitive registers
* Handle sensitive operations according to a policy.
    *  Ex: All VM disk IO is redirected to backing files in the host OS
    * VM cannot access the network at all, or only access a predefined set of remote IP addresses
* **Good**: Provides "complete" isolation (no guess instruction is directly executed on host hardware)
* **Good**: Virtual ISA can be different than the physical ISA
* **Bad**: Accurately emulating a modern processor is difficult
* **Bad**: This implementatino is VERY slow. (2x-3x) slower than direct execution
### What if we Assume That The Virtual ISA is The Same as the physical ISA? (Popek-Goldberg VMMs)
* Terminology: 
    * *Priviledged Instruction*: causes a trap if the processor isn't in priviledged mode
    * `invlpg`: invalidate a tlb entry
    * `mov %cr3, xxx`: Write to a control register
    * `inb`: read a byte of data from an input port
    * *Sensitive instruction*: access low-level machine stage that should be managed by control software
    * *Safe instructions* are not sensitive
* A VMM control program that is:
    * **Efficient**: All safe guest instructions run directly on hardware
    * **Omnipotent**: Only the VMM can manipulate sensitive state
    * **Undetectable**: A guest cannot determinin that it is running utop a VMM (red pilling, viruses might avoid detectability by not doing anything on a VM)
*  Does x86 support Popek-Goldberg virtualization?
    * x86 rings: 
        * ring 0: Most priviledged (kernel). Only ring 0 can invoke priviledged instructions
        * ring 1: 
        * ring 2: 
        * ring 3: Least priviledged
* An x86 PTE only has a single bit to indicate/supervisor!!! The MMU maps 0,1,2 to supervisor and ring 3 to user
* Intel originally thought that device drivers could be "isolated" here.
    * However, "isolated" drivers can still access ring 0 memory pages. Added segment based protection.
* Normal x86 setup: rings 1 and 2 go unused!!!!
* **Theorem**: A virtual machine monitor can be constructed for architectures in which every sensitive instruction is priviledged.
    * Guest app executes syscall to read from keyboard
    * Hardware generates trap which is handled by VMM
    * VMM vectors control to guest OS's interrupt handler
    * Guest OS tries to execute `inb` to read a key from keyboard
    * Hardware traps
    * VMM executes real `inb`
    * VMM places the result to where the guest OS expects it to be
    * VMM vectors control back to the guest OS
    * Guest OS does some processing and then does `iret/sysexit`
    * Hardware traps to VM
    * VMM returns to user-mode

* For many years, x86 had sensitive instructions that weren't priviledged!
* `push` can push a register value onto the top of the stack
* `%cs` register contains (among other things) 2 bits that represent the current priviledge level (i.e. the current ring)
*  `push %cs` -> look and see if it is running in the VM!!!! Detectable!
*  The `pushf/popf` instructions weren't PG virtualizable!
    * Read/write `%eflags` register using the value on the top of the stack!
    * Bit 9 of `%eflags` enables interrupts!
* To be virtualizable, `pushf/popf` should cause traps in ring 3. 
### Dynamic Binary Translation!!!
* Suppose that
    * VMM runs in Ring 0
    * Guest OS and apps run in Ring 3
* Allow guest user-level programs run atop hardware
* Translate sensitive-but-unpriviledged instructions into ones that trap to the VMM
* Example, replace `popf` with `int3`.
* However, Dynamic binary translation is complicated and hard to get right!!!
### Hardware-Assisted Virtualization
* Intel Virtualization Technology (Intel VT-x)

**Intel VT-x Sensitive Instruction**:
* VT-x make all sensitive instructions priviledged
* VT-x also creates a new priviledge bit, orthogonal to rings, to support virtualization
    * Root mode: Hypervisor code is running
    * Non-root mode: VM code is running
* In non-root mode:
    * Sensitive instructions invoked by Ring 0 (i.e. the guest OS) will trap to Root 0 root code (i.e. priviledged Hypervisor code)
* Sensitive instructions invoked by Rings 1--3 will trap to Ring 0 non-root mode, so that the guest OS can handle the misbehaving guest application
* In root mode, sensitive instructions executed by Rings 1--3 code will trap to Ring 0 root code (priviledged hypervisor code)

**Intel VT-x: Interrupts and Exceptions**
* When configuring a VM, the hypervisor decides if interrupts (e.g. timer interrupts, SSD interrupts) that occur during VM executing are vectored to:
    * Hypervisor code (via the hypervisor's IDT) 
    * Guest OS code (via the VM's IDT)
* Similarly, the hypervisor configures a VM to send exceptions to the hypervisor IDT or the VM IDT

**Intel VT-x Virtualizing a MMU**: 
* Goals: Allow a guest to believe that it controls the virtual-to-physical mapping
* Enable a hypervisor to actually control which guest pages live in RAM and where those pages live
* **Memory types in root mode**
    * Guest virtual memory: Visible to VM code
    * Guest physical memory: What the guest OS maps guest virtual memory to
    * Host physical memory

*Intel VT-x: Extended Page Tables*:
* In root mode, address translation works as normal
    * `%cr3` (controlled by the Ring 0 root-mode hypervisor) points to the page table that maps virtual addresses to physical ones
* The TLB caches virtual to physical mappings
* In non-root mode, VT-x uses an extended page table (EPT) register to tranlsate guest physical addresses to host physical addresses