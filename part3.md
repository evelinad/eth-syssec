# Part 3: Security of x86-based Systems
## Application Security
- Requirements
	- Launch-time Integrity: correct application was started/loaded
	- Run-time Isolation: no interference from malicious software/peripherals possible
	- Secure Persistent Storage
	- Code should have launch-time integrity and run-time isolation
	- Volatile data should have launch-time integrity and run-time isolation
	- Persistent data should be able to use a secure persistent storage
- Required functionality
	- Launch-time Integrity: integrity (e.g. hashes) verification of initial code and data
	- Run-time Isolation:
		- prevent unauthorised modification of code and data
		- prevent run-time attacks to modify execution flow
	- Secure Storage:
		- confidentiality
		- integrity protection
	- This security functionality must be secured too
	- Unclear where and how to implement the functionality

## Platform Overview
- Processor: contains one or more CPUs (called "cores" in this lecture)
	- MMU: Memory Management Unit
	- Programmable Interrupt Controller
	- Intel VMX: Virtual Machine Extensions
	- L1, L2 caches for each core
	- L3 cache shared by all cores
- Chipset: connects processor to memory and peripherals
- Peripherals: connected to the chipset through multiple bus-interfaces

## OS-Based Security
- Privilege Rings
	- Ring 0: Kernel
	- Ring 1: Device Drivers
	- Ring 2: Device Drivers
	- Ring 3: Applications
	- Lower number -> higher privileges
	- Nowadays only Ring 0 and Ring 3 are used
	- Device drivers part of the kernel now
	- Processor tracks Current Privilege Level (CPL) using two register bits
	- Used to limit access to privileged instructions and I/O ports
	- In the past used to protect kernel memory
- Paging-based Security: Security relevant data in page table entries
	- Supervisor bit: if set, page only accessible by ring 0 (isolates operating system from applications)
	- RW bits: allows to distinguish between read-only and writeable pages
	- Execution disable (ED) bit: if set, page isn't executable (e.g. prevents run-time code injection)
- DMA through IOMMU
	- Use IOMMU (Input/Output Memory Management Unit)
	- Controls DMA access to physical memory
	- Set up by operating system
	- Allows to make sure DMA access is not done to security relevant pages (page tables, kernel pages, etc.)

## TPM
see [TPM](tpm.md).

## Trusted Execution Environment (TEE)
- Smart Cards
	- e.g. with a Java Runtime
	- Dedicated and constrained TEE
- Trusted Execution Technology
	- e.g. TrustZone by ARM
	- System-wide TEE
- Software Guard Extensions
	- Concurrent TEE
- Dedicated powerful TEEs (e.g. by HP)

## Secure Execution Environment (SEE): Intel Trusted Execution Technology (TXT)
- Hardware components
	- CPU extended with new instruction: e.g. `SENTER` to enter SEE and `SEXIT` to exit SEE
	- Chipset contains additional SEE configuration registers
		- Adress Range Filtering: Used to setup the secure environment
		- Additional registers used to setup and use the secure environment
	- TPM used
		- PCR 16 to PCR 23 used
		- these are dynamically resettable PCRs, i.e. can be reset before system reboots
		- Access to PCRs over five memory pages governed by chipset that enforces access privilege levels
	- Cache-as-RAM support
- Software components
	- BIOS support needed: designates a single core that is allowed to execute `SENTER` and `SEXIT`, sets up DMA Protected Region, writes TXT registers
	- Authenticated Code Module: signed code module provided by Intel. Used to bootstrap a secure environment
	- Measured Launched Environment: The SEE environment. May also refer to the application running inside the secure environment
	- Operating System support needed: Adds Measured Launched Environment to DMA Protected Region, writes meta-data into TXT heap (e.g. entry-point)
- Authenticated Code Module is executed in cache (using Cache-as-RAM)
- `SENTER` steps
	1. OS disables: paging, all interrupts, executes `SENTER`
	2. BIOS' selected core: checks paging, interrupt status, requests other cores to sleep
	3. BIOS' selected core: fetches Authenticated Code Module into cache, checks signature, puts hash of it into one of the dynamically resettable registers
	4. Authenticated Code Module is execute (in cache!): checks location and integrity of Measured Launched Environment, hashes it and puts it into one of the dynamically resettable registers
	5. Measured Launched Environment starts execution, can wakeup other cores
- Important properties during measurements
	- During measurements operating system is suspended and can't interfere
	- Because all interrupts are disabled the same is true for peripherals
	- PCRs used only reset by the chipset (set to 0), chipset recognises `SENTER` when processor uses a special bus cycle

## ARM TrustZone
- Virtual processor with configurable system resources it can use
- Virtual processor is a special, secure, system running a trusted operating system
- Advantages: faster than a Smart Card, familiar software development model
- Trusted operating system
	- may verify integrity of application
	- may provide secure and persistent storage
- Hardware components are TrustZone aware
- Limitations
	- hard to implement a secure and trustworthy operating system
	- implementing interface to outside world also hard
	- each hardware/software piece must trust other pieces running in the secure environment

## Intel Software Guard Extensions (SGX)
- Scales well: each application runs in a separate Trusted Execution Environment (TEE)
	- compromising one of the applications in the TEE can't compromise other applications
	- compromised operating system not a problem
	- Eavesdropping on the Bus or malicious DMA access can't access sensitive parts
- No trust put into the operating system
	- Although it manages the TEE memory, it can't access it because it's encrypted
- Limitations
	- Overhead introduced by cryptographic functions used
	- Context switching costly
	- No peripheral access, e.g. no I/O

## Java Cards
- Uses Java Card Run-Time Environment (JCRE)
- Applications implemented as Applet, each Applet runs separated from other Applets
- Java has some nice security relevant properties: strongly typed, access control to attributes/methods, no pointers
- Applets separated by «Applet Firewall»: enforces access control through permissions and role-based rules
- Cryptographic support: e.g. integrity verification and authentication for Applets
- Each Applet has its own hierarchical filesystem
- Attacks still possible
	- Invasive attacks: memory readout, bus probing, etc.
	- Remote attacks: JCRE bugs, weak cryptography, etc.
	- Primary problem: hard resource constraints

## Runtime Attack Basics
- Exploit application's vulnerabilities
- Control-flow attack
- Code Injection: Adding new *node* to Control Flow Graph
	- attacker can execute arbitrary code
	- generally considered more powerful
- Code Reuse: Adding a new *path* to Control Flow Graph
	- attacker is limited to use of nodes already in Control Flow Graph
- Nowadays attackers use both techniques (code injection and reuse)
- Data Execution Prevention (DEP)
	- prevent execution of data located inside writeable memory location
	- widely adapted by operating systems and hardware

## Return-Oriented Programming (ROP)
- One example of ROP is «return-to-libc»
	- redirect execution to instructions inside shared libraries
	- libc is linked to nearly every application on Unix/Linux
	- Contains system calls and basic facilities (e.g. `open()`, `malloc()`, `system()`, etc.)
- Use only small pieces of larger instructions, all of them end with a `RET`
- Multiple such small sequences chained together are called *Gadget*
- Each Gadget does one specific task (e.g. load something, store something, branch, etc.)
- Adversary combines Gadgets

### Countermeasures
- Compiler Extension
	- Compiler adds security measures
	- adds performance overhead
	- Stack Canaries
		- Add so called *canary* values around control flow information and check it's still valid when control flow related information is used
		- Example: `PUSH` canary value at the beginning of a function, check value before `RET`
		- If adversary overwrites a control flow related information, it will also overwrite the canary
		- Works only if canary is not known to attacker, otherwise he can do an overwrite without changing the canary
- Programming Languages: Use languages that incorporate security relevant checks (e.g. boundary checks)
- Address Space Layout Randomization (ASLR)
	- randomize base address of code segments (stack, heap, dynamic libraries)
	- enabled in most operating systems
	- prevents attacker to do Return-Oriented Programming as the attacker does not know the memory address of the instructions
	- Code Injection harder as attacker must find out where the injected code is locates (its base address)
	- Typical problems:
		- Entropy is often too low, can be brute-forced
		- Disclosure attacks possible
		- not all code parts can be randomized, attacker can use the base addresses of the fixed parts
- Software Diversity
	- Create from the same source code multiple semantically equal executables
	- Exploits should work only on one of the executables
	- All shared libraries must also be diversified, otherwise attacker can use them to exploit the application
	- No secure and practical scheme available today for diversification to counter control flow attacks
- Binary Instrumentation
	- "Instrumentation" binds additional code to a program. Usually used for security and profiling purposes
	- Probe-Based Instrumentation: At program start code is rewritten/modified
	- JIT based instrumentation: Instrumentation code is added dynamically
		- Use shadow stack that holds expected return addresses
		- Check if return address is the same as on the shadow stack if a `RET` should be executed
	- usually introduces a performance penalty
