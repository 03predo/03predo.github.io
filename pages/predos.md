---
permalink: /predos
layout: page
title: predOS
page_title: >-
  predOS: An Operating System for the Raspberry Pi 1
---

![rpi1](assets/imgs/rpi1.jpg)


In this project I started designing and implementing an operating system for the Raspberry Pi 1. At this point I have implemented a UART driver, EMMC driver, FAT file system, and system calls such as fork, wait, execv, and signal, among others. The operating system has a shell that can be accessed via UART to spawn applications. The applications currently implemented are shell, echo, ps, and blink which are detailed later in this article. The source code for this project can be found [here](https://github.com/03predo/predOS/tree/main), and I used some scripts from [this](https://github.com/BrianSidebotham/arm-tutorial-rpi) repository to generate the image.

Specifications referenced in this article:

- [BCM2835 Peripheral Spec](assets/datasheets/bcm2835-peripherals.pdf)
- [SD Physical Layer Spec](assets/datasheets/Part1_Physical_Layer_Simplified_Specification_Ver3.01.pdf)
- [SD Host Controller Spec](assets/datasheets/PartA2_SD_Host_Controller_Simplified_Specification_Ver3.00.pdf)
- [Partition Table Spec](https://wiki.osdev.org/Partition_Table)
- [File Allocation Table Spec](https://wiki.osdev.org/FAT)
- [ARMv6 Reference Manual](assets/datasheets/ARMv6_reference_manual.pdf)

# Table of Contents
{:.no_toc}
* toc
{:toc}

# UART Driver

The mini UART peripheral on the BCM2835 provides a way for the host to communicate with the SOC. The implementation details for this peripheral can be found in section 2 of the BCM2835 Peripheral Spec. The peripheral triggers an interrupt when a byte is transmitted or received, there is a single 1 byte wide register that can be written to or read from to transmit or receive bytes. My driver implementation consists of two circular buffers, one for RX and TX. The TX buffer is filled by the kernel, on an interrupt the status register is checked and if the peripheral is ready to transmit then the byte at the current index in the buffer is written to the IO register and the index is incremented. The RX buffer is filled when the interrupt status register indicates a received byte. Applications can interact with this driver by reading and writing from the the standard input and standard output.

# EMMC Driver

The external mass media controller (EMMC) is a peripheral on the BCM2835 that acts as an interface to the SD card. Implementation details for this peripheral can be found in section 5 of the BCM2835 Peripheral Spec.

The EMMC is compliant with the SD Host Controller Specification. This spec describes a set of registers that allow the host to communicate with an SD card. While there are around 34 registers in the spec the key registers used to communicate with the card are the transfer mode, command, response, data, and interrupt status registers along with some control registers that are only used for initialization. The detailed description of the registers can be found in the SD Host Controller Spec, a general description can be found below.

- **Transfer Mode Register:** Configures the transfer mode. It indicates which direction data will be moving, and if the transfer is single or multi-block.
- **Command Register:** Configures the type of command being sent. It indicates what action the card is going to take, whether there is data being transferred, and what kind of response the command will receive. A write to this register also results in the command being sent by the host controller to the card.
- **Response Register:** Contains the reponse to the most recently completed command.
- **Data Register:** Used to transmit and receive data to and from the card.
- **Interrupt Status Register:** Contains various status flags that can be polled when waiting for an event such as a command being completed or the data register having new data to read.
- **Control Registers:** The Clock Control and Software Reset registers are used for initializing the card.

Note that the registers in the SD Host Controller spec are all 16 bits wide, whereas registers on the BCM2835 are 32 bits wide. This results in two 16 bit registers being concatenated into a single 32 bit register in the EMMC peripheral.

## Command Procedure

Communication between the card and the host is done by the host sending commands to the card, and the card sending back a response. To before writing to the command register, set `Argument Register 1` and `Argument Register 2` with any necessary arguments for the command. If the command is a data transfer command then fields in the `Transfer Mode Register` need to be configured to indicate the type of data transfer, and if the transfer is multi-block then the number of blocks must be written to the `Block Count Register`. The `Command Register` needs to be configured to indicate the command index, type of command, and the type of response, writing to this register will iniate the command so it must be the final step. Once the command is sent the `Interrupt Status Register` can be polled until the `Command Complete` field is set. Information about the commands and their responses can be found in the Physical Layer Spec.  

## Initialization Procedure

### 1. Resetting the Host Controller Circuit and Set Clock

On startup the host circuit needs to be reset. In the `Clock Control Register` set the `SD Clock Enable` and `Internal Clock Enable` fields to disable both clocks. Then in the `Software Reset Register` set the `Software Reset For All` field to reset the entire host controller circuit and wait for the reset to finish. Then use the `Clock Control Register` to set the clock frequency using the `SDCLK Frequency Select` field, then enable the internal clock and wait until it is stable by polling `Internal Clock Stable`, once stable enable the SD clock. During initialization the clock frequency should be set to the identification mode frequency which is 400kHz.

### 2. Transition from Idle to Standby State

During initialization the card must transition through various states before data transfers can be made. The state transition table in section 4.8 of the SD Physical Layer Spec describes the state transitions that are made when a command is sent to the card. A condensed version of the table, with only the necessary commands transitioning from idle to standby state, is shown below. The first column gives the command, and the preceeding columns give the state the card will transition to next while in the current state which is given by the column header. A "-" indicates that the command is invalid in the current state.

| COMMAND | IDLE | READY | IDENT | STANDBY | 
| :-----: | :--: | :---: | :---: | :-----: |
| GO_IDLE_STATE | idle | idle | idle | idle | 
| SD_SEND_OP_COND | ready | - | - | - | 
| ALL_SEND_CID | - | ident | - | - |
| SEND_RELATIVE_ADDR | - | - | standby | standby |

The first command we send is the GO_IDLE_STATE, this command will send the card into the idle state regardless of the previous state it was in.

Once in the idle state we send the SD_SEND_OP_COND command which describes the operating conditions of the host controller, this includes the  VDD voltage window and whether the host controller supports SDHC and SDXC. The response to this command is the OCR Register which is defined in section 5.1 of the Physical Layer Spec and describes the operating conditions the card is able to operate in and also indicates the power up status of the card. This command is repeatedly sent until the power up state bit indicates the power up routine is finished, this will result in the card transitioning to the ready state.

Once the card is powered up we send the ALL_SEND_CID command which returns the CID Register. This register contains various information about the card including the manufacturer ID, the product name, and the product serial number, this command will transition the card into the identification state.

Finally the SEND_RELATIVE_ADDR command is sent to retrieve the relative card address (RCA). The RCA is used to address which card is being used in a transfer and transitions the card into the standby state. If there are multiple cards in the system you can repeatedly send ALL_SEND_CID and SEND_RELATIVE_ADDR to obtain an RCA for each card.

### 3. Set Clock to Data Transfer Frequency

Once the card is in standby state the clock frequency should be set to the data transfer mode frequency which is 25MHz. This can be done by setting the `SD Clock Enable` field in the `Clock Control Register` to disabled and setting the new frequency in the `SDCLK Frequency Select` field. Wait until the internal clock is stable by polling `Internal Clock Stable` field, once stable enable the SD clock.

After these steps have been completed the card is ready to transfer data to and from the host.

## Data Transfer Procedure

### 1. Select a Card

The SELECT_DESELECT_CARD command is sent with the relative card address as the argument. This will transition the corresponding card into the transfer state.

### 2. Initiate the Transfer

Set the `Block Count Register` to the number of blocks in the transfer. Send either the WRITE_MULTIPLE_BLOCK or READ_MULTIPLE_BLOCK command. Once this command is complete, wait for the `Buffer Write Ready` or `Buffer Read Ready` fields in the `Interrupt Status Register` to indicate that transfer is ready to begin.

### 3. Transfer Data

Once the 'Buffer Write Ready` or `Buffer Read Ready` flags indicate the data port is ready these flags need to be cleared. Then you can write or read 32 bit words from the `Buffer Data Port Register`. Once you have written or read one block worth of bytes (512 bytes), you must wait for the `Buffer Write Ready` or `Buffer Read Ready` flags to indicate the next block is ready.

### 4. Finish the Transfer

Once all of the blocks have been transferred, you must wait for the `Transfer Complete` flag in the `Interrupt Status Register` to indicate the transfer is complete.

# File System

## Master Boot Record

The Master Boot Record (MBR) is placed in the first sector of the SD card and contains information about the partitions on the card. There are 4 partition entries in the MBR and each contain information about the partition such as the starting sector, the size in sectors, the file system ID, and whether the partition is in use.

## FAT File System

The File Allocation Table (FAT) file system organizes sectors on the card into files. This is done through three data structures, the BIOS parameter block, the file allocation table and directories. 

### BIOS Parameter Block

The BIOS parameter block contains metadata about the FAT filesystem including the number of sectors in a cluster, the size of the file allocation table, the number of entries in the root directory. Through this info we are able to determine the type of file allocation table (either 12-bit, 16-bit, or 32-bit) and the starting sector for both the file allocation table and the root directory.

### Directories

Directories are special files that contain a series of directory entries which describe files in the file system. Each directory entry contains the name of the file, file attributes, timestamps for creation date and last access time, the size of the file and the first cluster of the file. The first cluster of the file will act as an index into the file allocation table. As of right now I have implemented the feature for creating new directories so the only directory in the file system is the root directory.

### File Allocation Table

The file allocation table describes how the clusters on the card are being used by the files in the file system. The clusters that make up a file are stored in the table as a singly linked list and the head of that linked list (ie. the first cluster in the file) is stored in the directory entry of the file. The table is an array of entries, each of which represents a cluster on the card. The first cluster of the file acts as an index into the table, and the value stored at that index will be the next cluster in the file. Using that next cluster as an index into the table we can read the value at that index to find the next cluster in the file. You can continue this process until an entry in the table reads all 1s or 0xFFFF. The width of the entries is determined by fields in the BIOS parameter block, they are either 32-bit, 16-bit, or 12-bit. 

Below is a sample file allocation table for FAT16 (16-bit entries), the first two clusters (cluster 0 and cluster 1) are reserved so their values will always be 0xfff0 and 0xffff. Our first file begins at cluster 2 and ends at cluster 7, we have another file that begins at cluster 8 and ends at cluster 10 and a final file that begins at cluster 11 and ends at cluster 13. The remaining entires in the table are unused and set to 0.

| Address | +0x0   | +0x2   | +0x4   | +0x6   | +0x8   | +0xa   | +0xc   | +0xe   | 
| :-----: | :-----:| :----: | :----: | :----: | :----: | :----: | :----: | :----: |
| +0x0000 | 0xfff0 | 0xffff | 0x0003 | 0x0004 | 0x0005 | 0x0006 | 0x0007 | 0xffff |
| +0x0010 | 0x0009 | 0x000a | 0xffff | 0x000c | 0x000d | 0xffff | 0x0000 | 0x0000 |

## System Inode Table

The system inode table contains inodes for each of the currently open files. When a process opens a file an entry in the table is filled with the access flags, the file offset, and the files FAT directory entry. The index to this entry in the table is called a file descriptor and is returned to the process when the open system is called. Whenever a read or write is performed to the file the file offset is update and when the file is closed the modified directory entry is written back to the root directory and the entry in the inode table is free to be used for a different file.

# Memory Management

## Frame Allocation

To execute a process it must be loaded into memory, it is up to the kernel to find a suitable space in memory for process instructions and stack. To do this the kernel has a structure called the frame table. The frame table treats all of main memory as a series of 1Mb (2^20 byte) frames, the reason for this number will be explained in the virtual memory section. When a process is created the kernel will look through the frame table to see if there is any free frames and mark them as used for that process, when a process is destroyed these frames are marked as free.

## Virtual Memory

The biggest problem with having multiple processes is that at compile time they don't know what address they will be loaded into , the solution to this problem is using virtual memory. Virtual memory is simply a mapping of virtual addresses to physical addresses. With this mapping two different processes can access the same virtual address but each will have a different mapping for that address and thus be accessing two different physical addresses.

## Page Tables

On the ARM1176 the Memory Management Unit (MMU) is responsible for converting virtual addresses into physical addresses, a structure called page table is used to describe these mappings. The system page table contains the mappings for a series of pages also called sections. These sections are each 1Mb (2^20) which match the size of a single frame. The way a virtual address is mapped into a physical address is by taking the upper 12 bytes of the virtual address and using them as an index into the page table, the entry at that index will contain the upper 12 bytes of the physical address these upper 12 bytes are referred to as the section base address in the ARMv6 Reference Manual.

A diagram of this mapping is shown in figure B4-4 of the ARMv6 reference manual. The translation table base is the base address of the system page table, and the modified virtual address is the virtual address the process is using. The upper 12 bytes of the virtual address act as the table index. At that index there is a first-level descriptor which contains the section base address which is used with the lower 20 bytes of the virtual address to make the final physical address.

![](assets/imgs/b4_4_section_translation.png)

### Second Level Page Tables

If more fine mapping is required than instead of the system page table entry simply holding the section base address it can contain the address to a second level table which can have mappings for smaller pages within the larger section. A digram for this is shown in figure B4-7 of the ARMv6 reference manual. We can see that instead of the first-level discriptor containing the section base address it contains a page table base address. After the upper 12 bits of the virtual address the next 8 bits are used to index the second-level page table. This second level index contains the small page base address, using this base address and the remaining 12 bits from the virtual address we make the final physical address.

![](assets/imgs/b4_7_small_page_translation.png)

## Memory Layout

The diagram below shows both the physical memory layout and the virtual memory layout for a given process X. The exception vector is placed at address 0x00000000 and holds the addresses to the various exception handlers that are in the kernel text section. Following that are the page tables which need to be placed in a page that is mapped one to one and also need to be aligned on a certain byte boundary. The first page table is the root coarse page table which is a second level page table used to map the first frame of memory. We use a second level page table for this because we need the first few pages for the exception vector and page tables but also all programs default to the text segment starting at 0x8000 so the pages after that are needed for processes. After that is the system page table which contains the mappings for all of the frames in main memory. Then the kernel text segment is loaded in at address 0x8000, this is because the firmware on the Raspberry Pi will load the kernel image to this address. After this the frames are allocated for processes until frame 0x1f0 which is the start of the heap. The heap as of right now only is used by the standard library functions so there is no memory allocation and deallocation system in place. After the heap is the final frame in the system which is reserved for the kernel stack.

Processes are virtualy mapped for their text to start at 0x8000 and their stack to be the same as the physical address, kernel text is all mapped to addresses beyond 0xC000000.

![](assets/imgs/mem_layout.png)

# Process Management

The state of a given process is stored in its process control block (PCB). The PCB contains the process ID, stack pointer, signal handlers, and process state alongside fields to store its stack and text frames and information for when the process is blocked. The process can be in one of 6 states, the states are unused, ready, running, blocked, sleep, and signal. Unused processes are uninitialized, ready processes are ready to be scheduled by the kernel, the running process is the process currently executing, a blocked process is blocked waiting for some event, an asleep process is waiting for a certain timestamp to become ready, and a process in the signal state means a signal handler is being executed.

# System Calls

The system call interface provides a way for a process to call on the kernel to perform some function. The process can trigger a system call by executing the Supervisor Call (SVC) instruction. The SVC instruction has an argument that is used indicate which function the process wants to perform.

## System Call Procedure

When the SVC instruction is executed it triggers a software interrupt, this results in the software interrupt handler being called. In the software interrupt handler the CPU registers are stored on the stack of the currently running process. The argument of the SVC instruction can be retrieved using the link register which holds the address of the instruction to return to after the SVC instruction. Decrementing the link register will give the address of the SVC instruction and the argument can be decoded from the SVC opcode.

Based on the SVC argument the kernel will perform the system call requested by the process. Any arguments for the system call can be taken from the registers store on the stack. After the system call has been performed the kernel will look for a ready process and pop the registers off the stack and return the link register. If the system call has a return value then this is stored in the R0 register. To update the R0 register of the calling process the register value in the process stack can be overwritten and when the registers are popped off the stack this will result in the new value being placed in R0.

## File System Calls

System calls such as open, close, read, write, and lseek will directly execute file system functions on a given file. As of right now the file system will not cause a process to become blocked but simply wait for the operation to finish and return to the process. If the given file descriptor to read or write is STDIN or STDOUT this will result in UART driver functions being called. If the process is attempting to write to STDOUT then the buffer will be copyed to the UART TX buffer, if the process is attempting to read from STDIN this will result in the process being blocked and placed in the read queue. Once the specified number of bytes is received the kernel will copy from the RX queue into the process buffer and set the process to READY.

## Process Management System Calls

System calls such as fork, execv, wait, and exit all deal with the creation and destruction of processes.

### fork

The fork system call will result in the current process creating a child process with the same state. When this function is called the text and stack frames of the current process are copyed into two new empty frames, the parent process ID is stored in the child and the childs stack frame is virtually mapped to the stack frame of the parent. The fork function will return the process ID of the child in the parent process and in the child process it will return 0.

### execv

The execv system call will replace the current process with a new one based on a given file name and arguments. The given file is required to be an executable and linkable file (ELF). The ELF file and program headers are read and the appropriate segements are loaded into the current process text segment. The stack pointer is set to the top of the stack segment and the link register is set to the program entry point which is specified in the ELF file header. 

### wait

The wait system call will block the current process and place it in the wait queue. When the child process exits it will unblock the parent process and set its exit status as the return value to the function.

### exit

The exit system call cause the current process to be destroyed. It will notify its parent process of its exit status and then free up its stack and text frame and then transition to the unused state.

## Sleep System Call

The sleep system call will put the process in the sleep state until a certain timestamp is passed, then the process will transition back to the ready state. The yield system call will keep the process in the ready state but call the scheduler to see if there are any other ready processes that can be run

## Signal System Calls

Systems calls such as signal, raise, and kill all deal with sending signals between processes. 

### signal

The signal system call will register a signal handler for a given signal number. This handler is stored in the PCB of the process.

### raise

The raise system call will result in the signal handler being called for the given signal number. The kernel will store the current state and stack pointer of the current process and then push a new set of registers onto the stack that will be used by the signal handler. With the R0 register is set to be the signal number and the link register is set to be the exit system call. Then the process transitions into the signal state and the program counter is set to be the signal handler. In the exit system call the process state is checked to see if it was in the signal state, if so the stack pointer and process state are restored to their previous values and process returns to executing where it was after the raise call.

### kill

The kill system is similar to the raise system call except you supply the pid of a process and that processes signal handler will be called. This call can be used to send signals between processes as a form of inter process communication.

## Custom System Calls

I implemented two custom system calls ps and led.

### ps

This system call prints out information about the currently running tasks including their file names, process IDs, states, and uptime

### led

This system call allows a process to turn the onboard LED on and off.

# Applications

## init

This app is responsible for spawning the shell on startup and also acts as the idle process, so if all other processes are block this process will run.

## shell

This app allows the user to run other applications via UART. It will read from STDIN and spawn the given app with the given arguments and wait the given app to complete before accepting more inputs.

## echo

This app takes the arguments and writes them to STDOUT

## ps

This app calls the ps system call to list information about the active processes.

## blink

This system call can set a period for the onboard LED to turn on and off. On the first time this app is run it creates a daemon called blinkd, this process opens a file called blink.io where it stores the daemon process ID and the current period. On subsequent calls a process can read from blink.io to find the pid of the daemon and write the new period to the file. Once the write is complete the blink app sends a signal to the daemon to indicate the period is ready to be updated.

# Next Steps

This project is far from over and there are many features that I am excited to add. The first thing I need to work on is more unit tests and automated test infrastructure, this will require creating a way to load the kernel and applications over uart. After that I want to update the emmc driver to utilize the DMA and also update the file system to have directories. Once all of that is stable I want to start work on the USB driver and rendering simple graphics.


