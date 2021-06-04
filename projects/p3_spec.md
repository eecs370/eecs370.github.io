---
layout: spec
---

Project 3 EECS 370 (Spring 2021)
==============================
{: .primer-spec-toc-ignore }

<!-- TODO: update due dates -->
| Worth:             | 100 points                         |
| Assigned:          | Friday, June 4th, 2021   	      |
| Due:               | 11:55 pm, Friday, June 11, 2021  |


# 1. Purpose

This project is intended to help you understand in detail how a pipelined
implementation works.  You will write a cycle-accurate behavioral simulator
for a pipelined implementation of the LC-2K, complete with data forwarding and
simple branch prediction.

# 2. LC-2K Pipelined Implementation

## 2.1. Datapath

<!-- TODO: add hyperlink to lecture slides -->
For this project we will use the datapath from [Lecture 12](https://drive.google.com/file/d/1xaoG0dg8EUobqg_NdUkLQvk5_ouPSMGL/view?usp=sharing), "Introduction to Pipelining."

One difference between Project 3 and the pipelined processor described in
lecture is that we will add a pipeline register after the write-back stage
(the WBEND pipeline register).  This will be used to simplify data forwarding
so that the register file does not have to do any internal forwarding. It will
also mean that 3 noops will be required to avoid hazards instead of the 2
needed as discussed in class.

You will use a clocking scheme mimicking real-life processors (e.g., register
file and memory writes require the data to be present for the whole cycle).

## 2.2. jalr

You will not implement the jalr instruction from the LC-2K.  Taking out jalr
eliminates several dependencies.

## 2.3. Memory

We are keeping a simple representation of a memory unit. Just access memory
directly as an array (similar to [Project 1](https://eecs370.github.io/projects/p1_spec.html)).

Note in the typedef of `stateType` below that there are two memories: `instrMem`
and `dataMem`.  When the program starts, read the machine-code file into BOTH
`instrMem` and `dataMem` (i.e., they will initially have the same contents).
During execution, read instructions from `instrMem` and perform load/stores
using `dataMem`.  That is, `instrMem` will never change after the program starts,
but `dataMem` will change.  In a real machine, these two memories would be an
instruction and data cache, and they would be kept consistent.

## 2.4. Pipeline Registers

To simplify the project and make the output formats uniform, you must use the
following structures without modification to the existing variable names in
order to hold pipeline register contents. Note that the instruction gets passed
down the pipeline in its entirety.

```c
#define NUMMEMORY 65536 /* maximum number of data words in memory */
#define NUMREGS 8 /* number of machine registers */

#define ADD 0
#define NOR 1
#define LW 2
#define SW 3
#define BEQ 4
#define JALR 5 /* JALR will not implemented for Project 3 */
#define HALT 6
#define NOOP 7

#define NOOPINSTRUCTION 0x1c00000

typedef struct IFIDStruct {
	int instr;
	int pcPlus1;
} IFIDType;

typedef struct IDEXStruct {
	int instr;
	int pcPlus1;
	int readRegA;
	int readRegB;
	int offset;
} IDEXType;

typedef struct EXMEMStruct {
	int instr;
	int branchTarget;
	int aluResult;
	int readRegB;
} EXMEMType;

typedef struct MEMWBStruct {
	int instr;
	int writeData;
} MEMWBType;

typedef struct WBENDStruct {
	int instr;
	int writeData;
} WBENDType;

typedef struct stateStruct {
	int pc;
	int instrMem[NUMMEMORY];
	int dataMem[NUMMEMORY];
	int reg[NUMREGS];
	int numMemory;
	IFIDType IFID;
	IDEXType IDEX;
	EXMEMType EXMEM;
	MEMWBType MEMWB;
	WBENDType WBEND;
	int cycles; /* number of cycles run so far */
} stateType;

```

# 3. Problem

## 3.1. Basic Structure

Your task is to write a cycle-accurate simulator for the LC-2K.  We recommend
you start with the [Project 1 simulator](https://eecs370.github.io/projects/p1_spec.html#5-behavioral-simulator-40).  The main
modifications will be in the `run()` function.

At the start of the program, initialize the pc and all registers to zero.
Initialize the instruction field in all pipeline registers to the `noop`
instruction (0x1c00000).

`run()` will be a loop, where each iteration through the loop executes one
cycle.  At the beginning of the cycle, print the complete state of the machine
(you must use the `printState()` function at the end of this handout **WITHOUT
MODIFICATION**).  In the body of the loop, you will figure out what the new
state of the machine (memory, registers, and pipeline registers) will be at
the end of the cycle.  Conceptually, all stages of the pipeline compute their
new states simultaneously.  Since statements execute sequentially in C rather
than simultaneously, you will need two state variables: `state` and `newState`.
For clarity, `state` and `newState` refer to the C variable names, and state
without quotes refers to the English word meaning the condition of the datapath
at an instant of time. `state` will be the state of the machine while the cycle
is executing; `newState` will be the state of the machine at the end of the
cycle.  Each stage of the pipeline will modify the `newState` variable using the
current values in the "state" variable.  For example, in the ID stage, you might
have the following statement:

```
newState.IDEX.instr = state.IFID.instr (to transfer the instruction in the IFID register to the IDEX register)
```


In the body of the loop, you will use `newState` only as the target of an
assignment and you will use `state` only as the source of an assignment (e.g.,`
newState... = state...`).  `state` should never appear on the left-hand side of
an assignment (except for array subscripts), and `newState` should never appear
on the right-hand side of an assignment.

Your simulator must be pipelined.  This means that the work of carrying out an
instruction should be done in different stages of the pipeline as described in
lecture.  The execution of multiple instructions should be overlapped.  The ID
stage should be the only stage that reads the register file; the other stages
must get the register values from a pipeline register.  If it violates these
criteria, your program will get a 0.

Here's the main loop in `run()`.  Add to this code, but don't otherwise modify
it (and leave the comments as is) so we can understand your program more
easily.

```c
while (1) {

printState(&state);

/* check for halt */
if (opcode(state.MEMWB.instr) == HALT) {
    printf("machine halted\n");
    printf("total of %d cycles executed\n", state.cycles);
    exit(0);
}

newState = state;
newState.cycles++;

/* --------------------- IF stage --------------------- */

/* --------------------- ID stage --------------------- */

/* --------------------- EX stage --------------------- */

/* --------------------- MEM stage --------------------- */

/* --------------------- WB stage --------------------- */

state = newState; /* this is the last statement before end of the loop.
        It marks the end of the cycle and updates the
        current state with the values calculated in this
        cycle */
}
```

## 3.2. Halting

At what point does the pipelined computer know to halt?  It's incorrect to
halt as soon as a halt instruction is fetched because if an earlier branch was
actually taken, then the `halt` instruction could actually have been branched
around.

To solve this problem, halt the machine when a halt instruction reaches the
MEMWB register.  This ensures that previously executed instructions have
completed, and it also ensures that the machine won't branch around this halt.
This solution is shown above; note how the final `printState()` call before the
check for halt will print the final state of the machine.

## 3.3. Begin Your Implementation Assuming No Hazards

The easiest way to start is to first write your simulator so that it does not
account for data or branch hazards.  This will allow you to get started right
away.  Of course, the simulator will only be able to correctly run
assembly-language programs that have no hazards.  It is thus the
responsibility of the assembly-language programmer to insert noop instructions
so that there are no data or branch hazards.  This will require putting noops
in assembly-language programs after a branch and a number of noops in an
assembly-language program before a dependent data operation (it is a good
exercise to figure out the minimum number needed in each situation).

## 3.4. Finish Your Implementation by Accounting for Hazards

Modifying your first implementation to account for data and branch hazards
will probably be the hardest part of this assignment.

Use data forwarding to resolve most data hazards.  The ALU should be able to
take its inputs from any pipeline register (instead of just the IDEX
register).  There is no need for internal forwarding within the register file.
For this case of forwarding, you'll instead forward data from the WBEND
pipeline register. Remember to take the most recent data (e.g., data in the
EXMEM register gets priority over data in the MEMWB register).  **ONLY FORWARD
DATA TO THE EX STAGE (not to memory)**.

You will need to stall for one type of data hazard: a lw followed by an
instruction that uses the register being loaded.

Use branch-not-taken to resolve most branch hazards, and decide whether or not
to branch in the MEM stage.  This requires you to discard instructions if it
turns out that the branch really was taken.  To discard instructions, change
the relevant instructions in the pipeline to the noop instruction (0x1c00000).
Do not use any other branch optimizations, e.g., resolving branches earlier,
more advanced branch prediction, or special handling for short forward
branches.

# 4. Running Your Program

Your simulator should be run using the same command format specified in
Project 1:

```
./simulate program.mc > output
```

You should use the solution assembler from Project 1 to create the
machine-code file that your simulator will run (since that's how we'll test
it).  The solution assembler can be found at `/afs/umich.edu/class/eecs370/bin/asol`.

# 5. Test Cases

An integral (and graded) part of writing your pipeline simulator will be to
write a suite of test cases to validate any LC-2K pipeline simulator.  This is
common practice in the real world--software companies maintain a suite of test
cases for their programs and use this suite to check the program's correctness
after a change.  Writing a comprehensive suite of test cases will deepen your
understanding of the project specification and your program, and it will help
you a lot as you debug your program.

The test cases for this project will be short assembly-language programs that,
after being assembled into machine code, serve as input to a simulator.  You
will submit your suite of test cases together with your simulator, and we will
grade your test suite according to how thoroughly it exercises an LC-2K
pipeline simulator.  Each test case may execute at most 100 cycles on a
correct simulator, and your test suite may contain up to 20 test cases.  These
limits are much larger than needed for full credit.  See [Section 6]() for how
your test suite will be graded.
<!-- TODO: add hyperlink -->
Writing good test cases for this project will require more thinking than the
test suites in Project 1.  A pipeline simulator is much more complex than
the behavioral simulator, and the bugs that should be tested for are
correspondingly more complex.  Randomly choosing a few instructions is
unlikely to expose many pipelining bugs.  Think about how to test
systematically for pipeline-specific conditions, such as data forwarding,
branching, and stalling.  As you write the code for your simulator, keep notes
on what different conditions you've tested for (e.g., forwarding from different
stages).

# 6. Grading, Auto-Grading, and Formatting

**For spring 21 semester:**  To incentivize starting early,
students will receive 1 bonus point if they earn all of the available points for exposing instructor bugs **48-hours** before the posted deadlines.

We will grade primarily on functionality.  In particular, we will run your
program on various assembly-language programs and check the contents of your
memory, registers, and pipeline registers at each cycle.  Most of these
assembly-language programs will have hazards; a few will be hazard-free.
Since we'll be grading on getting the exact right answers (both at the end of
the run and being cycle-accurate throughout the run), it behooves you to spend
a lot of time writing test assembly-language programs and testing your
program.  Events must happen on the right cycle (e.g., stall the exact number
of cycles needed, write the branch target into the PC at exactly the right
cycle, halt at the exact right cycle, stalling only when needed).

To help you validate your project, your submission will be graded
automatically, and the result will be emailed back to you.  You may then
continue to work on the project and re-submit.  To deter you from using the
autograder as a debugger, you will receive feedback from the autograder only
for the first three submissions on any given day.  All subsequent submissions
will be silently graded.  Your final score will be derived from your overall
best submission to the autograder.

The results from the auto-grader will not be very illuminating; they won't
tell you where your problem is or give you the test programs.  The purpose of
the auto-grader is it helps you know to keep working on your project (rather
than thinking it's perfect and ending up with a 0).  The best way to debug
your program is to generate your own test cases, figure out the correct
answers, and compare your program's output to the correct answers.  This is
also one of the best ways to learn the concepts in the project.

The student suite of test cases for the simulator will be graded according to
how thoroughly they test an LC-2K pipeline simulator.  We will judge
thoroughness of the test suite by how well it exposes potential bugs in a
pipeline simulator.  The auto-grader will correctly assemble each test case in
your suite, then use it as input to a set of buggy simulators.  A test case
exposes a buggy simulator by causing it to generate a different answer from a
correct simulator.  The test suite is graded based on how many of the buggy
simulators were exposed by at least one test case.

You may submit your program as many times as you like.

Because all programs will be auto-graded, you must be careful to follow the
exact formatting rules in the project description:

1. Don't modify printState() at all.

2. There should be only ONE call to printState() in your program, which is the printState() shown in Section 3.1.  Do not put in any extra printState() calls (you can put these in for debugging, but take them out before submitting the program).

3. Initialize all values correctly.
    a. state.numMemory should be set to the number of memory words in the
    machine-code file.
    b. state.cycles should be initialized to 0.
    c. pc and all registers should be initialized to 0.
    d. the instruction field in all pipeline registers should be
    initialized to the noop instruction (0x1c00000).

    <!-- TODO: figure out how to indent -->

4. Check your program's output on the sample assembly-language program and output at the end of this handout.

5. Pay particular attention to what stage various operations are done in. For example, PC is incremented in the IF stage, so the IFID register should have PC+1.  Also, the sign-extender is in the ID stage, so the IDEX register should contain the value of offsetField AFTER calling convertNum().

6. Don't print the sequence `@@@` anywhere except in printState().

# 7. Turning in the Project

Use [autograder.io](https://autograder.io/) to submit your files.  You have been added as a student to 
the class, so you should see EECS 370 listed as a class.

1. pipeline simulator (part 3)
    a. Your simulator, a C program named `simulator.c`
    b. Suite of test cases (each test case is an assembly-language program in a separate file and end with `.as`)

    <!-- TODO: indent -->

Your simulator must be in a single C file.  We will compile your program on a
Linux workstation using

```console
gcc program.c -lm
```
so your program should not require additional compiler flags or libraries.

The official time of submission for your project will be the time the last file
is sent. If you send in anything after the due date, your project will be
considered late (and will use up your late days or will receive a zero).

# 8. Program Fragment

Here's the code for `printState()` and associated functions.  Don't modify this
code at all.

```c
void
printState(stateType *statePtr)
{
    int i;
    printf("\n@@@\nstate before cycle %d starts\n", statePtr->cycles);
    printf("\tpc %d\n", statePtr->pc);

    printf("\tdata memory:\n");
	for (i=0; i<statePtr->numMemory; i++) {
	    printf("\t\tdataMem[ %d ] %d\n", i, statePtr->dataMem[i]);
	}
    printf("\tregisters:\n");
	for (i=0; i<NUMREGS; i++) {
	    printf("\t\treg[ %d ] %d\n", i, statePtr->reg[i]);
	}
    printf("\tIFID:\n");
	printf("\t\tinstruction ");
	printInstruction(statePtr->IFID.instr);
	printf("\t\tpcPlus1 %d\n", statePtr->IFID.pcPlus1);
    printf("\tIDEX:\n");
	printf("\t\tinstruction ");
	printInstruction(statePtr->IDEX.instr);
	printf("\t\tpcPlus1 %d\n", statePtr->IDEX.pcPlus1);
	printf("\t\treadRegA %d\n", statePtr->IDEX.readRegA);
	printf("\t\treadRegB %d\n", statePtr->IDEX.readRegB);
	printf("\t\toffset %d\n", statePtr->IDEX.offset);
    printf("\tEXMEM:\n");
	printf("\t\tinstruction ");
	printInstruction(statePtr->EXMEM.instr);
	printf("\t\tbranchTarget %d\n", statePtr->EXMEM.branchTarget);
	printf("\t\taluResult %d\n", statePtr->EXMEM.aluResult);
	printf("\t\treadRegB %d\n", statePtr->EXMEM.readRegB);
    printf("\tMEMWB:\n");
	printf("\t\tinstruction ");
	printInstruction(statePtr->MEMWB.instr);
	printf("\t\twriteData %d\n", statePtr->MEMWB.writeData);
    printf("\tWBEND:\n");
	printf("\t\tinstruction ");
	printInstruction(statePtr->WBEND.instr);
	printf("\t\twriteData %d\n", statePtr->WBEND.writeData);
}

int
field0(int instruction)
{
	return( (instruction>>19) & 0x7);
}

int
field1(int instruction)
{
	return( (instruction>>16) & 0x7);
}

int
field2(int instruction)
{
	return(instruction & 0xFFFF);
}

int
opcode(int instruction)
{
	return(instruction>>22);
}

void
printInstruction(int instr)
{

	char opcodeString[10];

	if (opcode(instr) == ADD) {
		strcpy(opcodeString, "add");
	} else if (opcode(instr) == NOR) {
		strcpy(opcodeString, "nor");
	} else if (opcode(instr) == LW) {
		strcpy(opcodeString, "lw");
	} else if (opcode(instr) == SW) {
		strcpy(opcodeString, "sw");
	} else if (opcode(instr) == BEQ) {
		strcpy(opcodeString, "beq");
	} else if (opcode(instr) == JALR) {
		strcpy(opcodeString, "jalr");
	} else if (opcode(instr) == HALT) {
		strcpy(opcodeString, "halt");
	} else if (opcode(instr) == NOOP) {
		strcpy(opcodeString, "noop");
	} else {
		strcpy(opcodeString, "data");
    }
    printf("%s %d %d %d\n", opcodeString, field0(instr), field1(instr),
		field2(instr));
}

```

# 9. Sample Assembly-Language Program and Output

Here is a sample assembly-language program:
```
	lw	0	1	data1	$1= mem[data1]
	halt
data1	.fill	12345
```
and its corresponding output.  Note especially how halt is done (the `add 0 0 0`
instructions after the halt are from memory locations after the `halt`, which
were initialized to 0).  The `add 0 0 12345` instruction comes from the `.fill` line.
When converted to machine code, 12345 coming from a `.fill` looks the same as
`add 0 0 12345`. This instruction will propagate through the pipeline and the
addition will be performed. However, the register file is never modified due to
this instruction because when the halt instruction reaches the execution stage,
the program exits.

```
memory[0]=8454146
memory[1]=25165824
memory[2]=12345
3 memory words
	instruction memory:
		instrMem[ 0 ] lw 0 1 2
		instrMem[ 1 ] halt 0 0 0
		instrMem[ 2 ] add 0 0 12345

@@@
state before cycle 0 starts
	pc 0
	data memory:
		dataMem[ 0 ] 8454146
		dataMem[ 1 ] 25165824
		dataMem[ 2 ] 12345
	registers:
		reg[ 0 ] 0
		reg[ 1 ] 0
		reg[ 2 ] 0
		reg[ 3 ] 0
		reg[ 4 ] 0
		reg[ 5 ] 0
		reg[ 6 ] 0
		reg[ 7 ] 0
	IFID:
		instruction noop 0 0 0
		pcPlus1 -12973480
	IDEX:
		instruction noop 0 0 0
		pcPlus1 0
		readRegA 6
		readRegB 1
		offset 0
	EXMEM:
		instruction noop 0 0 0
		branchTarget -12974332
		aluResult -14024712
		readRegB 12
	MEMWB:
		instruction noop 0 0 0
		writeData -14040720
	WBEND:
		instruction noop 0 0 0
		writeData -4262240

@@@
state before cycle 1 starts
	pc 1
	data memory:
		dataMem[ 0 ] 8454146
		dataMem[ 1 ] 25165824
		dataMem[ 2 ] 12345
	registers:
		reg[ 0 ] 0
		reg[ 1 ] 0
		reg[ 2 ] 0
		reg[ 3 ] 0
		reg[ 4 ] 0
		reg[ 5 ] 0
		reg[ 6 ] 0
		reg[ 7 ] 0
	IFID:
		instruction lw 0 1 2
		pcPlus1 1
	IDEX:
		instruction noop 0 0 0
		pcPlus1 -12973480
		readRegA 0
		readRegB 0
		offset 0
	EXMEM:
		instruction noop 0 0 0
		branchTarget 0
		aluResult -14024712
		readRegB 12
	MEMWB:
		instruction noop 0 0 0
		writeData -14040720
	WBEND:
		instruction noop 0 0 0
		writeData -14040720

@@@
state before cycle 2 starts
	pc 2
	data memory:
		dataMem[ 0 ] 8454146
		dataMem[ 1 ] 25165824
		dataMem[ 2 ] 12345
	registers:
		reg[ 0 ] 0
		reg[ 1 ] 0
		reg[ 2 ] 0
		reg[ 3 ] 0
		reg[ 4 ] 0
		reg[ 5 ] 0
		reg[ 6 ] 0
		reg[ 7 ] 0
	IFID:
		instruction halt 0 0 0
		pcPlus1 2
	IDEX:
		instruction lw 0 1 2
		pcPlus1 1
		readRegA 0
		readRegB 0
		offset 2
	EXMEM:
		instruction noop 0 0 0
		branchTarget -12973480
		aluResult -14024712
		readRegB 12
	MEMWB:
		instruction noop 0 0 0
		writeData -14040720
	WBEND:
		instruction noop 0 0 0
		writeData -14040720

@@@
state before cycle 3 starts
	pc 3
	data memory:
		dataMem[ 0 ] 8454146
		dataMem[ 1 ] 25165824
		dataMem[ 2 ] 12345
	registers:
		reg[ 0 ] 0
		reg[ 1 ] 0
		reg[ 2 ] 0
		reg[ 3 ] 0
		reg[ 4 ] 0
		reg[ 5 ] 0
		reg[ 6 ] 0
		reg[ 7 ] 0
	IFID:
		instruction add 0 0 12345
		pcPlus1 3
	IDEX:
		instruction halt 0 0 0
		pcPlus1 2
		readRegA 0
		readRegB 0
		offset 0
	EXMEM:
		instruction lw 0 1 2
		branchTarget 3
		aluResult 2
		readRegB 0
	MEMWB:
		instruction noop 0 0 0
		writeData -14040720
	WBEND:
		instruction noop 0 0 0
		writeData -14040720

@@@
state before cycle 4 starts
	pc 4
	data memory:
		dataMem[ 0 ] 8454146
		dataMem[ 1 ] 25165824
		dataMem[ 2 ] 12345
	registers:
		reg[ 0 ] 0
		reg[ 1 ] 0
		reg[ 2 ] 0
		reg[ 3 ] 0
		reg[ 4 ] 0
		reg[ 5 ] 0
		reg[ 6 ] 0
		reg[ 7 ] 0
	IFID:
		instruction add 0 0 0
		pcPlus1 4
	IDEX:
		instruction add 0 0 12345
		pcPlus1 3
		readRegA 0
		readRegB 0
		offset 12345
	EXMEM:
		instruction halt 0 0 0
		branchTarget 2
		aluResult 2
		readRegB 0
	MEMWB:
		instruction lw 0 1 2
		writeData 12345
	WBEND:
		instruction noop 0 0 0
		writeData -14040720

@@@
state before cycle 5 starts
	pc 5
	data memory:
		dataMem[ 0 ] 8454146
		dataMem[ 1 ] 25165824
		dataMem[ 2 ] 12345
	registers:
		reg[ 0 ] 0
		reg[ 1 ] 12345
		reg[ 2 ] 0
		reg[ 3 ] 0
		reg[ 4 ] 0
		reg[ 5 ] 0
		reg[ 6 ] 0
		reg[ 7 ] 0
	IFID:
		instruction add 0 0 0
		pcPlus1 5
	IDEX:
		instruction add 0 0 0
		pcPlus1 4
		readRegA 0
		readRegB 0
		offset 0
	EXMEM:
		instruction add 0 0 12345
		branchTarget 12348
		aluResult 0
		readRegB 0
	MEMWB:
		instruction halt 0 0 0
		writeData 12345
	WBEND:
		instruction lw 0 1 2
		writeData 12345
machine halted
total of 5 cycles executed
```
