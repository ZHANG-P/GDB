# Hacking with GDB 
Now, we'd like to start an exciting topic. How they hack: buffer overflow and GDB Analysis. Yes, GDB is very power tool, which can assist in hacking. We introduce a very simple, but instructive example named buffer overflow. 

_overflow.c_:

```C
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
void granted();
int checkPasswd();
int checkPasswd()
{
    char passwd[16];
    printf("Enter your passwd: ");
    gets(passwd);
    
    if(strcmp(passwd, "passwd1"))
    {   
        printf("\nYou fail!\n");
    }   
    else 
    {   
        granted();
    }   

}
void granted(){
    printf("\nAccess granted\n");
    printf("You have gotten the privileges, and can do anything you like!! ");
    // Privileged stuff happens here.
    return ;
}
int main(){
    checkPasswd();
    return 0;
}
```
In _overflow.c_, the main function invoke the checkPasswd function. In checkPasswd() function, if the password we enter is correct, it will invoke granted() function and we will get the privileges to do anything. 
Then compile the source code:
```
$ gcc -fno-stack-protector overflow.c -o overflow 
```
Then we will see this warning information:
![warning](./figs/warning.png)
Here is a quote from `man gets`
>The gets() function cannot be used securely.  Because of its lack of bounds checking, and the inability for the calling program to reliably determine the length of the next incoming line.

Yes, just as you guess, we will make use of the dangerous gets() function to hack the _**overflow**_ program.

When we enter the correct password:
![correct password](./figs/correct.png)

When we enter a wrong password:
![wrong password](./figs/wrong.png)

Before we start to hack, it will be better for us to review the knowledge about stack.

## Procedure and Stack
A procedure call involves passing both data (in the form of procedure parameters and return values) and control from one part of a program to another. In addition, it must allocate space for the local variables of the procedure on entry and deallocate them on exit. Most machines, including IA32, provide only simple instructions for transferring control to and from procedures. The passing of data and the allocation and deallocation of local variables is handled by manipulating the program stack.

### Stack Frame Structure
IA32 programs make use of the program stack to support procedure calls. The machine uses the stack to pass procedure arguments, to store return information, to save registers for later restoration, and for local storage. The portion of the stack allocated for a single procedure call is called a stack frame. We should pay attention to that **the stack is growing from high memory address to low memory address** just as follows:
![Stack Frame](./figs/stack.png)

The topmost stack frame is delimited by two pointers, with register %ebp serving as the frame pointer, and register %esp serving as the stack pointer. The stack pointer can move while the procedure is executing, and hence most information is accessed relative to the frame pointer.
Suppose procedure P (the caller) calls procedure Q (the callee). The arguments to Q are contained within the stack frame for P. In addition, when P calls Q, the **return address** within P where the program should resume execution when it returns from Q is pushed onto the stack, forming the end of P’s stack frame. The stack frame for Q starts with the saved value of the frame pointer (a copy of register %ebp), followed by copies of any other saved register values. Procedure Q also uses the stack for any local variables that cannot be stored in registers. 
As described earlier, the stack grows toward lower addresses and the stack pointer %esp points to the top element of the stack. Data can be stored on and retrieved from the stack using the push and pop instuctions. **Space for data with no specified initial value can be allocated on the stack by simply decrementing the stack pointer %esp by an appropriate amount**. Here we shall help you review these two stack operation instructions with two examples:

- push: pushing a double-word value onto the stack involves first decrementing the stack pointer(%esp) by 4 and then writing the value at the new top of stack address. 
- pop: popping a double-word value involves reading from the top of stack location and then incrementing the stack pointer by 4.

### Transferring Control
Here, we mainly review three very important instructions.

- call
	- Call instruction has a target indicating the address of the instruction where the called procedure starts.
	- The effect of a call instruction is to **push a return address** on the stack and jump to the start of the called procedure
- leave: the leave instruction can be used to prepare the stack for returning. It first move the value from %ebp to %esp, then pop the stack to %ebp. 
- ret: the ret instruction pop the **return address** from caller's stack frame to %eip, at which execution will resume when the called procedure returns. 

Recall that the gets() function has no bound checking, and it will read what we input in a line to the empty space(named buffer) on the caller procedure's stack frame no matter how long the input is. So, for _**overflow**_ program, if our input is longer than the buffer, and overlap the **return address** on the top of main() procedure's stack frame with the memory address where calling the granted() function. So, when the checkPasswd() function returns, the ret instruction will pop this value to %eip, then whether the password we enter is correct or not, the granted function will be invoked. Now we have known the method to hack this program, let's fire it up with the powerful tool, GDB.

## Disassembling with GDB

We start up debugging with `gdb ./overflow`. Then we can use `info func` to see those funtions, which are involved. 
![info func](./figs/infofunc.png)

In order to get the **return address**, which the call instruction will push to the stack when calling checkpasswd() function, we disassamble the main() function using `disassembly main` function. After the checkPasswd() function returns, the execution will resume at the instruction in the red rectangle. So 0x0804851 will be the value of **return address**, which we'd like to overlap.
![disassemble main](./figs/disassemble.png)

To disassemble the checkPasswd() function, we use `disassemble` command again. Here, we get the memory address of calling granted function is 0x080484ea, with which we want to overlap **return address**. For inspecting the location of buffer on the stack, we set a breakpoint at memory location 0x080484b8 where calling gets() function. 
![disassemble checkPasswd](./figs/disassembleCheck.png)

In order to set a breakpoint at a specific memory address, we can use `break *` followed by a memory address. 
![break*](./figs/breakAddress.png)

After setting breakpoint, we `run` to reach first breakpoint. Here, we take `x/20x $esp` command to print the stack content from top to bottom. Enter `help x` in the GDB prompt, you can view the help information of command `x`. And we find the location of **reture address**, which was in the red rectangle, the memory address is 0xbffff00c.
![x/20x](./figs/x20x.png)

Then `nexti` to step one instruction but not step into gets() function. After we entering a sequence of characters "BBBBCCCC"(four 'B's + four 'C's, '0x42 and '0x43' denote 'B' and 'C' respectively in hex system), we `x/20x $esp` again. The characters overlap the memory space which is in green rectangles. 
![x/20x](./figs/x20x1.png)

Till now, we can know that the buffer start from memory address $0xbfffefe0, and we want to overlap **return address**(0x08048521) with memory address of calling granted() function(0x080484ea). So we need a sequence of 32 characters, in which the last 4 characters combined should be equal to 0x080484ea to overlap the **returnn address**.
For convenience, we can use following command in terminal to generate 32 characters in a file named attack.txt.
```
python -c 'print "C"*28 + "\xea\x84\x04\x08"' > attack.txt
```

Note that we input the bytes of 0x080484ea inversely, because our platform is based on little endian intel processor. 

Finally, `./overflow < attack.txt`, redirect the stdin to attack.txt. 
Bingo! Hacking done.
![hacking done](./figs/hackingDone.png)








 