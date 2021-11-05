# EECS 765 Project 4

Author: Jace Kline 2881618

## Introduction

The purpose of this programming assignment is to develop and execute a local buffer overflow attack against the Winamp application running on Windows 7. This attack involves utilizing the fact that the exception handler chain (SEH chain) lives on the user-level stack in Windows. We exploit this fact to overwrite the pointer to the appropriate exception handler function, ultimately hijacking the control flow and executing our own inserted shellcode.

## Setup

This exploit requires the Windows 7 VM provided on the PA4 instructions page. It is assumed that the Winamp and netcat programs are installed on this VM. The first step is to extract the ZIP file holding our exploit contents and moving the mcvcore.maki file to the "C:\Program Files\Winamp\Skins\Bento\scripts" directory on the Windows VM. Optionally, to recreate the MAKI file from the exploit script, one can run `perl exploit.pl > mcvcore.maki`.

## Running the Exploit

Before running the exploit, we assume that the environment is as described in the previous section. The first step is to open an administrator CMD shell on the Windows VM. Following this, start a netcat listener to listen to the reverse TCP connection initiated by our malicious shellcode. We use port 4444 as the port in our attack. Run the following command to start this netcat listener:

```sh
nc -nlvvp 4444
```

Next, open the Winamp application on the victim machine. Within the application GUI, click on the top left button and navigate to Skins > Bento. Clicking on this Bento option will result in the exploit being executed and the reverse TCP shell being created. Check the listener shell to verify that the shell was initiated. We show the approximate output of a successful attack in the image below.

<img src="https://github.com/jace-kline/eecs765-p4/raw/main/exploit.png" width="50%"/>

## Developing the Exploit

Our exploit hinges on the idea of overwriting a record in the SEH chain (exception handler chain) that lives on the stack in Windows operating systems. Each SEH record in this SEH chain has two subsequent 4-byte words of data. The first word is the address of the next record in the SEH chain. The second word is the address of the exception handling procedure that is called when this specific SEH record handles the exception. The idea behind the attack is to create a buffer overflow that causes an exception to be thrown and subsequently the correct SEH record to be overwritten.

### Malicious Input Structure

The malicious input structure used in our attack 

### Malicious Input Parameters

For this exploit, we require the following parameters:

* System architecture
* The length of the 'matching_pattern' buffer
* Global Offset Table (GOT) address of free()
* Address of the exploitable 'matching_pattern' buffer referenced in `getscore_heap.c`

For this attack, the RedHat 8 machine uses the x86 architecture. This implies little-endianness and a 4-byte word length.

For consistency with the sample exploit seen in class, we declare that the length of the 'matching_pattern' buffer shall be 128 bytes, which implies that the 'name' buffer shall be 111 bytes.

Our exploit requires that we overwrite an address of a function used in the victim program to hijack the execution of the program and jump to our shellcode written into the heap buffer. This is the 'what' address referenced in the exploit. By examining the source code, we choose the free() function since it is executed in the program after the call to `free(matching_pattern)`, which performs the overwrite. We use the command `objdump -R getscore_heap` to list the GOT functions and their addresses and we extract the address listed for free().

To perform the overwrite of the free() GOT entry, we must also obtain the address of the start of our injected code, which occurs at the start of the 'matching_pattern' buffer + 8 bytes. We use the `gdb` tool on the `getscore_heap` program to achieve this. We set a breakpoint after the allocation of the buffer in code and then output its address. We also automate this process in our exploit script.

### Generating the Malicious Input

To generate the malicious input and perform the exploit, we have provided a wrapper script called `exploit.sh`. The contents of this script are seen below.

```bash
#!/bin/bash

# compile exploit.c and getscore_heap.c if they aren't already
gcc -g exploit.c -o exploit
gcc -g getscore_heap.c -o getscore_heap

# get parameters for exploit programmatically
BUF_LEN=128
GOT_FREE=0x`objdump -R getscore_heap | grep free | awk '{print $1}'`
BUF_ADDR=`gdb -batch -x gdbcommands.txt 2>/dev/null | grep '$1' | awk '{print $3}'`

# run the exploit with the parameters instantiated
./exploit $BUF_LEN $GOT_FREE $BUF_ADDR
```

The execution of this script involves three main steps. The first is to compile the `exploit.c` and `getscore_heap.c` files to the binary executables `exploit` and `getscore_heap`, respectively (with debugging enabled). The second is to automatically obtain the 3 parameters needed for running the compiled `exploit` executable, namely the buffer length, the GOT entry address to overwrite, and the target buffer address to overflow. The last step involves running the `exploit` executable with the obtained parameters from step 2. This exploit executable generates the input buffers to send to the victim `getscore_heap` program and runs the victim program with those buffers. This `exploit` program is a modification of the `heap_exploit` program provided by Dr. Bardas. The modifications simply involve the division of the inputs correctly between the 'name' and 'ssn' buffers as discussed earlier.

## References and Collaborations

### References

* [GDB Man Pages](https://man7.org/linux/man-pages/man1/gdb.1.html)

### Collaborations

I did not collaborate with any other students on this programming assignment.
