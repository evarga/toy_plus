# Introduction
This is a teaching material aimed to demonstrate [bootstrapping](https://en.wikipedia.org/wiki/Bootstrapping) using the [TOY](https://introcs.cs.princeton.edu/java/62toy/) imaginary virtual machine (VM). You may want to read [this](https://blog.ervinvarga.com/2025/01/explaining-high-level-concepts-via-low.html) blog before proceeding further. It describes the rationale behind choosing TOY and how to setup the development environment.

The following topics are covered in this unit:

- Why *bootstrapping* is such a powerful concept in modern computing.
- The illustration of bootstrapping by implementing a TOY+ simulator in TOY that is more powerful than the original machine.
- How to design an evolvable system whilst preserving backward and forward [compatibility](https://en.wikipedia.org/wiki/Compatibility).

# The TOY+ Machine Design
The idea is to create a new machine that augments the [instruction set of TOY](https://introcs.cs.princeton.edu/java/62toy/cheatsheet.txt). The table below lists the new instructions supported by TOY+.

| Category | Disassembly | Description |
| -------- | ----------- | ----------- |
| logical | <code>R[s] <-> R[t]</code> | Swaps the content of the specified registers. |
| logical | <code>R[d] <- ~R[t]</code> | Stores in R[d] the complement of R[t]. |
| transfer | <code>R[s]&nbsp;->&nbsp;M[--R[t]]</code> | Pushes R[s] onto stack. The R[t] is decremented before writing into memory. |
| transfer | <code>R[d]&nbsp;<-&nbsp;M[R[t]++]</code> | Pops the top of the stack into R[d]. The R[t] is incremented after reading from memory. |

The following drawing depicts the layered design where the TOY+ VM simulator appears as just another TOY program. Apparently, it should still be capable of running old TOY programs alongside those that use the new instructions. On the other hand, all TOY+ programs should contain logic to immediately stop execution if run on the original TOY VM (more on this later).

```
+-------------+   +--------------+
| TOY program |   | TOY+ program | - uses the expanded VM
+-------------+   +--------------+
   |    \                 |
   |     \                |
   |      \               |
   |   +---------------------+
   |   |  TOY+ VM Simulator  | - more powerful than the underlying VM
   |   +---------------------+
   |              |
   |              |
+--------------------+
|  TOY VM Simulator  | - could run on top of another VM, like JVM
+--------------------+
```

## Extending the Instruction Set
To preserve backward compatibility, it is not acceptable to introduce new opcode formats. One viable extension point is the `halt` instruction that only uses the first 4 bits to encoding an operation. The other bits are ignored by the TOY VM[^1]. Hence, the new instructions are encoded as follows:

- **swap**: 01st
- **not**: 02dt
- **push**: 03st, where R[t] plays the role of a stack pointer
- **pop**: 04dt, where R[t] plays the role of a stack pointer

Notice that this design permits old programs to run unchanged on our new VM. New programs should start with the `0100` instruction, that tries to exchange R[0] with itself. This operation will stop the program on the old VM (it will appear as `halt`) whilst on the new one it will have no effect.

The TOY+ VM and a simulated program behaves as a composite, i.e., the VM halts when a simulated program halts. This is exactly the situation when a JVM executes a Java program. From the perspective of an operating system, they are inside a single process.

# Sample TOY+ Program
Let us revisit the TOY program presented in [this](https://blog.ervinvarga.com/2025/01/explaining-high-level-concepts-via-low.html) blog. Thankfully to TOY+'s built-in support for working with pushdown stacks the new [code](samples/ruler.toyp) is considerably shorter. Below is the fully commented and disassembled version:

```
program Ruler Function
// Test client for the ruler function.
00: 0100   swap R[0] <-> R[0]               fail-fast if run on old VM
01: 8AFF   read R[A]                        n = read from stdin
02: FF04   R[F] <- PC; goto 04                            
03: 0000   halt                                                     

function ruler
// Input: R[A] = n     
// Return address: R[F]
// Output: prints the relative lengths of the subdivisions on a ruler.               
// Temporary variables: R[1], R[2], R[3].
04: 7101   R[1] <- 0001                                       
05: 420A   R[2] <- R[A]                               
06: 7A20   R[A] <- 0020                     R[A] <- top of the pushdown stack

// Entry point into an inner function.
// R[2] = current level (depth)
07: 03FA   R[F] -> M[--R[A]]                push return address                                         
08: 032A   R[2] -> M[--R[A]]                push current level                      
09: 2321   R[3] <- R[2] - R[1]                             
0A: C310   if (R[3] == 0) goto 10           test for base case   
0B: 2221   R[2] <- R[2] - R[1]                            
0C: FF07   R[F] <- PC; goto 07              recursive call to inner function
0D: 1321   R[3] <- R[2] + R[1]              R[3] <- level (R[2] + 1)
0E: 93FF   write R[3]                       print level
0F: C009   goto 09                          tail recursion

// Base case when level is 1.
10: 92FF   write R[2]                       print 1
11: 042A   R[2] <- M[R[A]++]                pop current level                                  
12: 04FA   R[F] <- M[R[A]++]                pop return address                             
13: EF00   goto R[F]                        return
```

The TOY+ VM simulator accepts as input a [memory dump](https://en.wikipedia.org/wiki/Core_dump), which is essentially a contiguous sequence of words in hexadecimal format. The **size** and **relative starting address** appears at front the of the memory content. For the sake of completeness here it is in format expected by our simulator:

```
14
0
0100
8AFF
FF04
0000
7101
420A
7A20
03FA
032A
2321
C310
2221
FF07
1321
93FF
C009
92FF
042A
04FA
EF00
```

If your code starts at another address (instead of 0), then you should precede it with necessary placeholder words, so that the relative start address is correct when counted from the beginning of the core dump.

## Steps to Run the Sample Program on the TOY+ VM Simulator
1. Launch the [Visual X-TOY Simulator](https://lift.cs.princeton.edu/xtoy/) program (I recommend using the Jar application).
2. Load the [TOY+ VM Simulator](src/toyplus_vm_simulator.toy) using the *File->Open...* command.
3. Enter the debug mode by selecting *Mode->Debug Mode*. I also advise you to configure the performance parameters to speed up simulation (see the option *Change Settings...* in the lower left part of the UI).
4. Press the *Run* button.
5. The standard input related tab will start blinking, which is a signal that some input is expected there. Press *Open...* and choose the sample [code](samples/ruler.toyp).
6. After each input you should continue execution by pressing *Run* again.
7. Now our simulated program expects its own input, which is the argument *n*. Enter 4 (or some other number of decent magnitude) and press *Add*.
8. Press *Run* again.
9. You will see output generated by our ruler program on the standard output tab.

You should never forget that all values are expected in hexadecimal format!

# Conclusion
You have witnessed an ability to empower the underlying machine by writing a simulator on top of it that constitutes an improved version of itself. In our case, the TOY+ VM simulator is fully written in TOY and run on a TOY simulator. Programs that use the extended instruction set are leaner and more maintainable, since less code is needed for the same effect. This boosts developers' productivity since less lines of code are needed to achive the equivalent effect compared to the old machine.

You are encouraged to further expand the instruction set of devise a completely new machine. Of course, TOY's 256 sized memory limits what can be done, but even this TOY+ VM showcases how much can be fit even in this miniscule machine. 


[^1]: It turns out that letting those bits linger around and be ignored by the TOY VM was not a good design choice. It should have been better to mandate that `halt` is only 0000 and announce that those unused bits are for future use and should not be set. Luckily, most current TOY programs use 0000 as a `halt` instruction, but 0F3E would have the same effect.