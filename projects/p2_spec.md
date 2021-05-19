---
layout: spec
---

Project 2 EECS 370 (Spring 2021)
==============================

| Worth:                   | 100 Points                           |
|--------------------------|--------------------------------------|
| Assigned:                | Wednesday, May 19th, 2021            |
| Part 2A Due:             | 11:55 PM EDT, Tuesday, May 25th      |
| Part 2L Due:             | 11:55 PM EDT, Thursday, June 3rd     |

# 0. Starter Code
We provide [starter code for the linker](https://drive.google.com/drive/folders/1rVM_hiMtvIgNjShquiL2eTIfMEx2oNmz?usp=sharing). For the assembler, you should build on top of the Project 1A Assembler. 

# 1. Purpose


The purpose of this project is to help you understand the assembling and linking process, which we can utilize to create multi-file LC-2K projects, and to help you understand how procedure calls work in assembly language.

# 2. Problem

```
[[assembly files]] -assembler--> [[object files]] -linker--> [executable]
```

This project has two parts. In the [first part](#3-assembler-30), you will write a program that assembles an assembly file into an object file. In the [second](#4-linker-40), you will write a program to link object files into one executable consisting of machine code, which your project 1 simulator will be able to run.

In [Project 1a](https://eecs370.github.io/projects/p1_spec.html#4-lc-2k-assembly-language-and-assembler-40), you wrote an assembler which took an LC-2K assembly file as input and produced an executable file as output. This approach is fine if all the code needed is contained in one file, but what happens if we want to use other pieces of code? Libraries contain functions that make coding easier. Splitting code into multiple files encourages modularity and organization. Multiple files are also important for large projects; if you modify one file, you only need to re-assemble that one and then link everything together. Now that we have a better understanding of translation software, we can create a separate assembler and linker.


# 3. Assembler (40%)


Your new assembler will take in a single assembly file (see [section 3.1](#31-assembly-files)) as input and output a single object file (see [section 3.2](#32-object-file-format)).

## 3.1. Assembly Files


### 3.1.1 Assembly File Format


Assembly language programs will be of the same format as those from Project 1, with a few extra restrictions. 


The first part of the assembly file must contain only assembly instructions. The second part should contain only `.fill` assembler directives. For example, suppose an assembly file is composed of M instructions and N `.fill`’s. Lines 0 to (M-1) contain actual instructions, and lines M to (M+N-1) contain `.fill`'s, with no mixing between them.


LC-2K files may now use global symbolic addresses, which means we must now distinguish between local and global labels. The scope of a local label is the file the label is defined in. The scope of a global label is all object files linked together (more on this in [part 2l](#4-linker-40)). Because of this, different object files can use local labels with the same name and still be linked together. 


Local symbolic addresses must be defined at assembly time. However, a global symbolic address can be undefined at assembly time.  It is assumed that undefined global labels are defined in another file to be linked at compile time, so they should be temporarily resolved as address 0 in the text and data segments. Defined symbolic addresses should be resolved exactly as they were in Project 1.


You can assume assembly files max out at 65536 total instructions and data, although we’ll test you on much, much less than that.


### 3.1.2 LC-2K Peculiarities Part 1


Firstly, local labels will start with a lowercase letter `a…z` while global labels start with a capital letter `A…Z`. This is unique to LC-2K as a way to distinguish between local and global labels.


Secondly, if a branch instruction contains a symbolic address, the label it refers to must be a locally defined label. This label can be either a local or global label. Branching to another file (undefined global label) is bad style and makes linking needlessly difficult. A programmer should use jalr in this case. Keep this in mind when considering what to add to the Symbol and Relocation Tables.


Thirdly, in LC-2K, loading or storing to an absolute address no longer makes sense. The locations of data and text within the final executable file will likely be different than in the original object file, leading to unintended execution. While this isn’t something we will enforce with error checking, it is recommended that labels are used when dealing with loads and stores. In reality, there are reasons to use absolute addressing: memory mapped IO for example (if you’re curious about this, take EECS 373 **shameless plug**).  If you come across a label with a constant offset, just assemble as you would in Project 1.


Fourthly, local labels do not require symbol table entries. However, a local symbolic address does need a relocation table entry as the address of the local label might change. These addresses can be fixed by calculating the new local label location during linking.


### 3.1.3 Assembly File Format Summary


In summary, assembly file formatting rules are:

1. Do not mix instructions with directives (`.fills`)
2. Instructions come first
3. Directives come second
4. Defined symbolic addresses are resolved exactly as they were in the Project 1 assembler
5. Undefined global symbolic addresses are temporarily resolved as address 0
6. Local labels start with `a…z` and must be defined at assembly
7. Global labels start with `A…Z` and can be undefined at assembly
8. Branches cannot use undefined global symbolic addresses
9. Loads and stores should use symbolic addresses (but are not 
required to)


## 3.2 Object File Format


Object files will contain the following sections in the following order:
* `Header`
* `Text`
* `Data`
* `Symbol` table
* `Relocation` table


*** Refer to lecture for a detailed explanation of each section. ***


*Table 1: Object file sections*

| Section Name  |      Number of lines    |      Description       |
| --------------| ----------------------- | ---------------------- |
| `Header` | Fixed: 1 | The `Header` contains the size, in  lines, of the sections to follow. Sizes are listed in the following order, each separated by a space: `Text`, `Data`, `Symbol` table, `Relocation` table. |
| `Text` | Variable: `t` <br> `t` = # of instr. | Each line in the `Text` segment consists of a single machine code instruction, assembled in the same way as instructions in Project 1. |
| `Data` | Variable: `d` <br> `d` = # of .fills | The `Data` segment contains data stored by assembler directives, one word of data per line. |
| `Symbol` table | Variable: `s` <br> `s` = # of global labels + # of Unresolved global symbolic addresses | Each line in the `Symbol` table consists of a global label, one letter (T/D/U) corresponding to `Text`, `Data`, and Undefined respectively, and a line offset from the start of the T/D section (0 if the letter was ‘U’). Each value separated by a space, in that order. Each symbol should only appear once in the symbol table, even if it is used multiple Times. Entries can appear in any order. |
| `Relocation` | Variable: `r` <br> `r` = # of symbolic addresses used | Each line in the `Relocation` table consists of a line offset from the start of the T/D section (whichever section the symbol was used in), an opcode, and a label. Each separated by a space, in that order. Entries can appear in any order. |



#### IMPORTANT FORMATTING NOTES:
{: .primer-spec-toc-ignore }


1. Assembly code in text should be assembled EXACTLY as it was for project 1. This means symbolic addresses are resolved the same, with the exception of undefined global symbolic addresses which are temporarily assembled as 0.

2. Offsets in the Symbol and Relocation Tables indicate the line offset of the label from the start of the either the Text or Data section (whichever the section the label appears in). 

For example, the symbol table entry `Foo D 0` indicates the label `Foo` is defined on the zeroth line in the Data section. The relocation table entry `4 lw Foo` indicates the symbolic address `Foo` is used on the fourth line (zero indexed) of the Text section by a `lw` instruction.

<!-- TODO: Check that bullets line up properly -->


## 3.3 Error Checking


Your assembler should catch the following errors in assembly files:
1. Duplicate defined labels (same local or global label within one assembly file)
2. Undefined local symbolic address
3. `beq` using an undefined global symbolic address
4. `offsetFields` that don't fit in 16 bits
5. Unrecognized opcodes


Your assembler should `exit(1)` if it detects an error and `exit(0)` if it finishes without detecting any errors. Your assembler should NOT catch simulation-time errors, i.e. errors that would occur at the time the assembly-language program executes (e.g. branching to address -1, infinite loops, etc.).


## 3.4 Assembly example


Please see [section 9 example 2a](#example-2a).


## 3.5 Running Your Assembler


Write your program to take two command-line arguments. The first argument is the filename where the assembly-language program is stored, and the second argument is the filename where the output (the object file) is written. For example, with a program name of `assemble`, an assembly-language program in `program.as`, the following would generate an object file `program.obj`:

```console
./assemble program.as program.obj
```


Note that the format for running the command must use command-line arguments for the file names (rather than standard input and standard output). Your program should store only object files in the format specified above. Any deviation from this format (e.g. extra spaces or empty lines) will render your machine-code file ungradable. Any other output that you want the program to generate (e.g. debugging output) can be printed to standard output.


## 3.6 Test Cases


The test cases for the assembler part of this project will be short assembly-language programs that serve as input to an assembler. You will
submit your suite of test cases together with your assembler, and we will grade your test suite according to how thoroughly it exercises an assembler. Each test case may be at most 50 lines long, and your test suite may contain up to 20 test cases. These limits are much larger than needed for full credit (the solution test suite is composed of 5 test cases, each < 10 lines long). See [section 7](#7-grading-auto-grading-and-formatting) for how your test suite will be graded.


Hint: the example assembly-language program (Example 2a) in [section 9](#9-sample-test-cases) is a good case to include in your test suite, though you'll need to write more test cases to get full credit.  Remember to create some test cases that test the ability of an assembler to check for the errors in [section 3.3](#33-error-checking).


# 4. Linker (60%)


Now that you’ve written an assembler to create object files, you need a way to link these files together. In this part of the project, you will write a linker to combine multiple object files into a single executable.
This final executable will be backwards compatible with the simulator from project 1.


## 4.1 LC-2K linker description


Your linker should be able to take an arbitrary number of object files as input. It will concatenate all text and data segments within each object file, creating one unified executable. Segments should be combined in the order they appear as arguments. The combined text section should be placed before the combined data section. Then, for each object file, the linker iterates through their relocation table. For each relocation entry, the linker iterates through all symbol table entries to locate the label and fix the reference. The final executable will be a machine code file. 


## 4.2 What about `main()`?


You might be asking yourself, what will be executed first? Shouldn’t there be a `main()` function or label? 


To simplify the process of linking and simulating, LC-2K code is executed starting at the first line of in a machine code file. In order to specify what object file should execute first, ordering of the linker’s arguments is needed.

```console
./linker file_0.obj file_1.obj machine_code.mc
```


In the above example, `file_0.obj` contains the first instructions to be executed. The resulting machine code file should be laid out as follows:


```
———————— machine_code.mc ————————
        <file_0.obj TEXT>
        <file_1.obj TEXT>
        <file_0.obj DATA>
        <file_1.obj DATA>
```


For more information on the linker’s command line arguments, please see [section 4.7](#47-linker-example). For more information on how linker’s actually handle this, see [section 4.4](#44-lc-2k-peculiarities-part-2).


## 4.3 Stack Label


As discussed in lecture, programs build up stack frames as they execute. The stack is important for storing data that can’t fit within a machine’s registers, such as stack frames and local data. As seen in the below example, this is done by using a global label `Stack`. 


 
Here is a small LC-2K program that uses a subroutine call. It takes
an argument in register 1 and calls a subroutine to compute the quantity `4*input`. Register 1 is used to pass input to the subroutine; register 3 is used by the subroutine to pass the result back. The current top-of-stack (first empty location) is given by `Stack` + register 5.
 
```
–––––––– main.as ––––––––

        lw         0        1      input        r1 = memory[input]
        lw         0        4      SubAdr       prepare to call sub4n. r4=addr(sub4n)
        jalr       4        7                   call sub4n; r7=return address r3=answer  
        halt
input   .fill      10                        


–––––––– sub4n.as ––––––––

sub4n   lw          0       6       pos1        r6 = 1
        sw          5       7       Stack       save return address on stack
        add         5       6       5           increment stack pointer
        sw          5       1       Stack       save input on stack
        add         5       6       5           increment stack pointer
        add         1       1       1           compute 2*input
        add         1       1       3           compute 4*input into return value
        lw          0       6       neg1        r6 = -1
        add         5       6       5           decrement stack pointer
        lw          5       1       Stack       recover original input
        add         5       6       5           decrement stack pointer
        lw          5       7       Stack       recover original return address
        jalr        7       4                   return.  r4 is not restored.
pos1    .fill       1
neg1    .fill       -1
SubAdr  .fill       sub4n                         contains the address of sub4n
```
 
The stack array starts at the implicit label `Stack` and extends to larger addresses, which is why the linker inserts the `Stack` label as the last line in the final executable..

In LC-2K, the `Stack` label is a special label that should not be defined by any object file, but it can be used as a symbolic address. This label is inserted by the linker and should refer to the line after the last piece of data in the data segment. For example, if there are M instructions and N pieces of data in the final executable, the linker should resolve the symbolic address, `Stack`, as (M + N). This allows the stack to grow without affecting the instructions or data.


## 4.4 LC-2K Peculiarities Part 2


Programming languages often specify where to begin executing. In reality, a linker typically inserts an object file into the linking process. This inserted code appears first and jumps to a specified function to begin executing the program, among doing other things. The LC-2K method of ordering files during the linking process to indicate what to execute first is a simplification.


LC-2K also lacks a proper function call instruction that jumps to labels. It instead jumps to registers that hold function addresses (so the register is a function pointer). This means that a function can have a local label, yet still be accessible from other files, so long as the function pointer is global. Linking should still succeed in this case.


Additionally, LC-2K’s use of the `Stack` label doesn’t reflect how all assembly languages use the stack. ARM, for example, has special instructions such as `push` and `pop` that directly interface with the stack, providing a layer of abstraction to assembly programmers. The stack is typically allocated by an operating system that passes the stack pointer to an executing program.


## 4.5 Error Checking


Your linker should catch the following errors:

1. duplicate defined global labels 
2. undefined global labels
3. `Stack` label defined by an object file


Your linker can assume that any object file used as input is properly formatted.


## 4.6 Tip - Local Labels


Fixing local symbolic addresses during linking can be tricky, since we don’t have symbol table entries associated with them. It might help to store certain data for each file read in: text size, data size, text starting location (in final mc), and data starting location (in final mc). By also storing which file each relocation table entry is in, you should have all the data needed to adjust each local symbolic address. 


Actually fixing a local symbolic address in the relocation involves several steps. First, identify which section of the file the label is in, either text or data. Second, parse the original symbolic address value from the instruction referenced by the relocation entry. Fix this value by adding an offset to the address, to account for the new location of the local label.


## 4.7 Linker Example
Please see [section 9 example 2l](#example-2l).


## 4.8 Running Your Linker


Write your program to take N command-line arguments, where N >= 2. The first argument is the object file to execute first, arguments 2 through N-1 are additional object files (these are not required), and the Nth argument is the filename where the machine code output is written. For example, with a program name of `linker` and an assembly-language program in `prog_1.obj` and `prog_2.obj`, the machine code file `prog.mc` will be generated:

```console
./assemble prog_1.obj prog_2.obj prog.mc
```


The number of object files your linker must be able to link together is between 1 and 6. If a program is self-contained within one object file, your linker should still be able to translate it into a machine code file. We will not test you on linking more than 6 object files.


Note that the format for running the linker must use command-line arguments for file names (rather than standard input and standard output). Your program should store only machine-code in the format specified above. Any deviation from this format (e.g. extra spaces or empty lines) will render your machine-code file ungradable. Any other output that you want the program to generate (e.g. debugging output) can be printed to standard output.


## 4.9 Test Cases


Test cases for the linker part of this project will be short, valid assembly-language programs that, after being assembled into object files, serve as input to a linker. You will submit a suite of test cases together with your linker, and we will grade your test suite according to how thoroughly it exercises an LC-2K linker. Each test assembly file may be at most 50 lines long, and your test suite may contain up to 20 test cases. A test can contain no more than 6 assembly files to be linked together. These limits are much larger than needed for full credit. See [Section 7](#7-grading-auto-grading-and-formatting) for how your test suite will be graded.


A naming scheme is needed to specify what test assembly files should be linked together. A single “test” refers to a group of 1 or more assembly files to be linked together. The naming scheme is as follows.


```
<test name>_<{0, …, N}>.as
```


All tests with the same test name will be assembled and linked together (do not include angled brackets in the test name). An underscore character, ‘_’, separates the test name and the assembly file’s number (do not include angled brackets or curly brackets in the number). Assembly files within the same test should be numbered starting at zero, with the zeroth assembly file being the first code to be executed.


The following testcases:

```
test_0.as test_1.as test_2.as anotherTest_0.as anotherTest_1.as 
```


Will be assembled and then linked by the autograder as follows:


```console
./linker test_0.obj test_1.obj test_2.obj test.mc
./linker anotherTest_0.obj anotherTest_1.obj anotherTest.mc
```


DO NOT use more than one underscore in your test case names. We will not grade your test case if you do. 


Remember to create some test cases that test the ability of a linker to check for the errors in [Section 4.5](#45-error-checking).



# 5. Assembly-Language Function to Compute Combination(n,r)  (Removed for Spring)


# 6. Compiling the Project


Your code will be compiled with the GCC compiler using the C99 standard. The following bash command compiles `program.c` and writes the executable into program. You are allowed to use any standard C libraries which compile with the specified flags below.

```console
gcc -std=c99 program.c -o program
```


# 7. Grading, Auto-Grading, and Formatting


We will grade primarily on functionality, including error handling, correct assembly, and comprehensiveness of the test suites.


To help you validate your project, your submission will be graded automatically, and the result will be available on the website. You may then continue to work on the project and re-submit. To deter you from using the autograder as a debugger, you will receive feedback from the autograder only for the first **FIVE SUBMISSIONS** for each project part on any given day. All subsequent submissions will be silently graded. Your final score will be derived from your overall _best submission_ to the autograder.


The feedback from the autograder will not be very illuminating; it won't tell you where your problem is or give you the test programs. The purpose of the autograder is to let you know that you should keep working on your project (rather than thinking it's perfect and ending up with a 0). The best way to debug your program is to generate your own test cases, figure out the correct answers, and compare your program's output to the correct answer. This is also one of the best ways to learn the concepts in the project.


The student suites of test cases will be graded according to how thoroughly they test both the assembler (for [part 2a](#3-assembler-30)) and linker (for [part 2l](#4-linker-40)). We will judge thoroughness of the test suites by how well it exposes potential bugs.


For the assembler test suite, the auto-grader will use each test case as input to a set of buggy assemblers. A test case exposes a buggy assembler by causing it to generate a different answer from a correct assembler. 


For the linker test suite, the auto-grader will first assemble the test files and use them as input to a set of buggy linkers. A test case exposes a buggy linker by causing it to generate a different answer from a correct linker. Test cases must use the naming scheme specified in [section 4.9](#49-test-cases).


The test suites are graded based on how many of the buggy assemblers / linkers were exposed by at least one test case. This is known as `mutation testing` in the research literature on automated testing.


# 8. Turning in the Project


Use [autograder.io](https://autograder.io) to submit your files.  You have been added as a student to the class, so you should see EECS 370 listed as a class.


Here are the files you should submit for each project part:
```
1. assembler (part 2a)
    a. C program for your assembler called "assembler.c"
    b. suite of test cases (each test case is an assembly-language program in a separate file, ending in ".as")

2. linker (part 2l)
    a. C program for your linker called "linker.c"
    b. Suite of test cases (each test case is a set of assembly-language programs using the naming scheme specified in [section 4.9](TODO:). and ending in ".as"

s"
e    halt
five    .fill       5


–––––––– subone.as ––––––––

subOne  lw          0       2       neg1
        add         1       2       1         
        jalr        7       6
neg1    .fill       -1
SubAdr  .fill       subOne

```

And here are the corresponding object files after running the following lines of code:

```console
./assembler main.as main.obj
./assembler subone.as subone.obj
```


NOTE: Text within parenthesis should not be included in your assembler’s or linker’s output.

```
–––––––– main.obj ––––––––

6 1 1 2                 (Header)
8454150                 (Text)
8650752
23527424
16842753
16842749
25165824
5                       (Data)
SubAdr U 0              (Symbol Table)
0 lw five               (Relocation Table)
1 lw SubAdr


–––––––– subone.obj ––––––––

3 2 1 2                 (Header)
8519683                 (Text)
655361
25034752
-1                      (Data)
0
SubAdr D 1              (Symbol Table)
0 lw neg1               (Relocation Table)
1 .fill subOne

```
This is `main.obj`:

```
6 1 1 2
8454150
8650752
23527424
16842753
16842749
25165824
5
SubAdr U 0
0 lw five
1 lw SubAdr

```
This is `subone.obj`:
```
3 2 1 2
8519683
655361
25034752
-1
0
SubAdr D 1
0 lw neg1
1 .fill subOne

```
**NOTE: Be careful when copying and editing these examples**

### Example 2l
This example uses the object files from example 2a. Here is the machine code produced after the linking process:

```console
./linker main.obj subone.obj count5.mc
```

```
–––––––– count5.mc ––––––––

8454153             (main.as TEXT)
8650763
23527424
16842753
16842749
25165824
8519690             (subone.as TEXT)
655361
25034752
5                   (main.as DATA)
-1                  (subone.as DATA)
6

```

This code can be simulated using your project 1 simulator.

**NOTE: Be careful when copying and editing these examples**


# 10. Linker Starter code


Referenced starter code is meant to help you read in and parse object files. It is probably a good idea to break it up into different functions, but is a good place to get started.


NOTE: Please see starter file.
