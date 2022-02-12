- [CS/COE 1541 - Introduction to Computer Architecture](#cs-coe-1541---introduction-to-computer-architecture)
- [Introduction](#introduction)
  * [Description](#description)
  * [Processor Design](#processor-design)
- [Building and Running](#building-and-running)
  * [Environment Setup](#environment-setup)
  * [Directory Structure and Makefile Script](#directory-structure-and-makefile-script)
  * [Program Output](#program-output)
  * [Creating Performance Plots](#creating-performance-plots)
- [Configuration Files and Trace Files](#configuration-files-and-trace-files)
  * [Configuration Files](#configuration-files)
  * [Trace Files](#trace-files)
- [Your Tasks](#your-tasks)
  * [Task 1: Enforcing Stalls and Flushes on Hazards](#task-1-enforcing-stalls-and-flushes-on-hazards)
    + [Structural hazards](#structural-hazards)
    + [Data hazards](#data-hazards)
    + [Control hazards](#control-hazards)
  * [Task 2: Enabling Optimizations on the Hazards](#task-2-enabling-optimizations-on-the-hazards)
  * [Source Code](#source-code)
  * [Submission](#submission)
- [Resources](#resources)
  * [GitHub Primer](#github-primer)
  * [Debugging C](#debugging-c)
  * [Using DDD through remote X11 forwarding](#using-ddd-through-remote-x11-forwarding)
    + [For Windows 10](#for-windows-10)
    + [For MacOS](#for-macos)
    + [For Linux](#for-linux)
    + [Establishing SSH connection with X11 forwarding](#establishing-ssh-connection-with-x11-forwarding)
  * [Using DDD by installing on local machine](#using-ddd-by-installing-on-local-machine)
    + [For Windows 10 WSL or Linux](#for-windows-10-wsl-or-linux)
    + [For Mac](#for-mac)
  * [DDD hang issue solution](#ddd-hang-issue-solution)


# CS/COE 1541 - Introduction to Computer Architecture
Spring Semester 2022 - Project 1

Please accept Project 1 on **GitHub Classroom** using the following link:
https://classroom.github.com/a/jREHYDcC

* DUE: Mar 16 (Wednesday), 2022 4:30 PM 

# Introduction

## Description

The goal of this project is to create a software simulator for a simplified 2-wide in-order processor architecture.  So why do architects build simulators before creating the actual hardware design?

1. Simulators allow them to test out the performance of various designs and parameters before building the real chip.  Designing a chip, verifying it, and fabricating it is hugely expensive and time consuming.  But software simulators allow hundreds of different combinations of designs and parameters to be tried out by simply modifying the simulator configuration file.  These designs are tested against multiple benchmarks to measure their relative performances. Some popular ones are [SPEC Benchmark](https://www.spec.org/benchmarks.html), [LINPACK Benchmark](http://www.netlib.org/benchmark/hpl/), and [NAS Parallel Benchmark](https://www.nas.nasa.gov/publications/npb.html).  Benchmarks that reflect the applications that the processor is targeted for are prioritized.

1. Simulators not only provide information about speed but also other considerations, one major one being power consumption.  The end of Dennard Scaling and the subsequent Power Wall has made measuring power consumption an indispensable part of processor design.  Power models such as the [SPICE Model](https://en.wikipedia.org/wiki/SPICE) allow architects to estimate the power use of circuits.  Simulators can also emulate the effects of variability in the processor manufacturing process and how it effects the end product.  Simulators are also used to measure the reliability of processors by emulating what happens when temperature hotspots form or bit flips are caused by cosmic radiation.

1. Simulators also expose a lot of corner cases and bugs in a design before going to the fab.  Building a simulator forces an architect to implement in some detail (albeit in software) the ideas he/she had and see it in motion, not just on a piece of paper.  For example, if your simulator falls into an infinite loop, it is very likely your hardware design will too!  Hardware designs are increasingly parallel nowadays, and just like parallel software dataraces, hardware can also have hardware races.  A prime source of bugs is the cache coherence protocol, that is, how do you keep caches in multiple cores coherent with each other when the cores are updating data in parallel.  The higher the fidelity of the simulator to the actual hardware design, the higher the chance of finding defects.

You will build your simulator with similar goals in mind.  You will identify the structural, data, and control hazards present in your processor design, implement the necessary pipeline stalls and flushes required to avoid them, and measure the impact they have on performance.  Next, you will implement optimizations to the design that can avoid those stalls and flushes and evaluate them too.

## Processor Design

The simplified 2-wide processor pipeline that you will simulate has the following basic structure:

<img alt="Pipeline" src=Project1_diagram.png width=700>

The processor can fetch two instructions at a time, decode two instructions at a time, and also writeback the results two at a time, which is why it is called a 2-wide processor.  For the EX stage, instructions are routed to two different execution units depending on the instruction type.  Lw (load word) and sw (store word) instructions are routed to the lw/sw EX unit and all instructions other than lw/sw are routed to the ALU/Branch EX unit.  Lw/sw instructions also pass through a MEM unit that performs the load or store.  ALU/Branch instructions don't require a MEM unit but they pass through an "empty" pipeline stage nonetheless so that both lw/sw and ALU/Branch instructions can perform WB at the same stage.  Having WB, or register writeback, at the same stage simplifies processor design (for example, allowing less write ports in the register file).

The fact that each individual EX unit can process only certain types of instructions can lead to situations where the pipeline is not filled (bubbles).  For example, if the ID stage is filled with two ALU/Branch instructions then at the next clock cycle, only one of those instructions will be routed to the ALU/Branch EX unit and the lw/sw EX unit must sit idle.  Also, the remaining ALU/Branch instruction needs to wait one more cycle before being routed.  The act of routing an instruction to the correct EX unit in computer architecture parlance is called **issuing** an instruction.

Now, given the above situation, you may be tempted to dynamically reorder instructions such that the ID stage is not filled with two instructions of the same type so that the processor can issue to both EX units at each cycle.  This is called dynamic scheduling, as we learned in class.  With dynamic scheduling, the processor would queue up many instructions at the ID stage (called the **issue queue**) such that the processor has a large window of instructions with a high probability that there is an instruction to issue for each EX unit at each cycle.  This type of processor is called an **out-of-oder** processor since it has the ability to issue instructions out of original program order.  But in order to do this, the processor needs to do complex dynamic data dependence analysis to make sure that the reorderings are legal.  We cannot properly implement an out-of-order processor within the time constraints of this class, so we will implement an in-order processor instead.

An **in-order** processor issues instructions strictly in program order.  Thus, it avoids violating data dependences in the program in a simplified way.  That means the issue queue operates like an actual queue (first-in-first-out).  The queue order is the program order.  In this organization, it does not make sense to have a queue length longer than the issue width, since the processor will at most be looking at two instructions at a time.  Hence, the queue length is two, equal to ths width of the processor.

# Building and Running

## Environment Setup

The project is setup to run with the g++ compiler (GNU C++ compiler) and a Make build system.  This system is already in place on the departmental thoth machine (thoth.cs.pitt.edu).  If you have a similar setup on your local computer, please feel free to use your machine for development.  Otherwise, you need to log in to thoth.cs.pitt.edu which may involve some setup.  Here are the steps you need to take:

1. Most OSes (Windows, MacOS, Linux) comes with built-in SSH clients accessible using this simple command on your commandline shell:
   ```
   ssh USERNAME@thoth.cs.pitt.edu
   ```
   If you want a more fancy SSH client, you can download Putty, a free open source terminal:
   https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
   Connect to "thoth.cs.pitt.edu" by typing that in the "Host Name" box.  Make sure that port is 22 and SSH is selected in the radio button options.

2. Once connected, the host will ask for your Pitt SSO credentials.  Enter your password.

3. Create and go to a directory of your choice (or you can stay at your default
   home directory) and then clone your GitHub Classroom repository:

   ```
   git clone <your GitHub Classroom repository HTTPS URL>
   ```

   This will ask for your Username and Password.  Username is your GitHub
account username, but Password is not your password.  Password authentication
on GitHub has been deprecated on August 2021, so now you have to use something
called a Personal Authenication Token (PAT) in place of the password.  Here are
instructions on how to create a PAT:
https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token

## Directory Structure and Makefile Script

Here is an overview of the directory structure:

* config.c / config.h : Functions used to parse and read in the processor configuration file.
* **CPU.c / CPU.h** : Implements the five stages of the processor pipeline.  The code you will be **modifying**.
* **five_stage.c** : **Main function**. Invokes the five stages in CPU.c at every clock cycle.
* **five_stage_solution** : **Reference solution binary** for the project.
* generate_plot.plt : GNUPlot script to generate the plot PDF file from the data.
* generate_plot.py: Python script to extrace performance data from results in tabular form.
* Makefile : The build script for the Make tool.
* trace.c / trace.h : Functions to read and write the trace file.
* trace_generator.c : Utility program to generate a trace file of your own.
* trace_reader.c : Utility program to read and print out the contents of a trace file in human readable format.
* confs/ : Directory where processor configuration files are.
* **diffs/** : Directory where **differences** between outputs/ and outputs_solution/ are stored.
* outputs/ : Directory where outputs after running five_stage are stored.
* outputs_solution/ : Directory where outputs produced by five_stage_solution are stored.
* plot_confs/ : Directory where processor configurations for the plot generation are.
* plots/ : Directory where outputs after running five_stage are stored for plot generation.
* plots_solution/ : Directory where outputs after running five_stage_solution are stored for plot generation.
* traces/ : Directory where instruction trace files used to test the simulator are stored.

In order to build the project and run the simulations, you only need to do 'make' to invoke the 'Makefile' script:

```
make
```

The output should look like:

```
g++ -c -g -Wall -I/usr/include/glib-2.0/ -I/usr/lib64/glib-2.0/include/ five_stage.c
g++ -c -g -Wall -I/usr/include/glib-2.0/ -I/usr/lib64/glib-2.0/include/ config.c
g++ -c -g -Wall -I/usr/include/glib-2.0/ -I/usr/lib64/glib-2.0/include/ CPU.c
g++ -c -g -Wall -I/usr/include/glib-2.0/ -I/usr/lib64/glib-2.0/include/ trace.c
...
```

If successful, it will produce the binaries: five_stage, trace_generator, and
trace_reader as well as results of the simulation using all combinations of 4
configuration files and 8 trace files.  A configuration file represents a
processor design and a trace file contain instructions from a micro-benchmark,
so you will be in effect simulating 4 processor designs on a benchmark suite.
You can generate your own traces using the trace_generator and put it inside
the traces/ directory or create a new configuration inside the confs/
directory, and they will be incorporated into the results automatically by the
Makefile script.  The results are stored in the outputs/ directory and also
side-by-side diffs with the outputs_solution/ directory are generated and
stored in the diffs/ directory.  When you debug the program, you will find
these side-by-side diffs useful.

If you only wish to build your C files and not run the simulations, just do
'make build' to invoked the 'build' target in the 'Makefile' script:

```
make build
```

If you wish to remove all files generated from your five_stage implementation (object files and experiment output), invoke the 'clean' target:

```
make clean
```

If you wish to remove all generated files (including ones generated from five_stage_solution), invoke the 'distclean' target:

```
make distclean
```

Optionally, you can also run your simulator on more sizable benchmarks.  I have 4 short and 2 long trace files (sample1.tr, sample2.tr, sample3.tr, sample4.tr) and (sample_large1.tr, sample_large2.tr). These files are accessible at /afs/cs.pitt.edu/courses/1541/long_traces and /afs/cs.pitt.edu/courses/1541/short_traces. But these are not incorporated into the Makefile script because they take significantly longer to run.  When you do run these on five_stage, I recommend you do not have the -v (verbose) or -d (debug) flags on or the simulations will take too long and the output may overflow your disk space.

## Program Output

You are given a program, five_stage.c, which reads a trace file (a binary file containing a sequence of executed instructions) and simulates a 5 stage pipeline ignoring any control and data hazards. It outputs the total number of cycles needed to execute the instructions in the trace file and, also calculates the IPC (Instructions Per Cycle):

```
./five_stage -t traces/sample.tr -c confs/2-wide.conf
```

The output should look like:

```
+ Number of cycles : 804
+ IPC (Instructions Per Cycle) : 1.2438
```

The IPC is the measure of performance for the processor design.

If the -v (verbose) option is given on the commandline, instructions that enter the pipeline (IF) and instructions that exit the pipeline (WB), along with corresponding clock cycles, are printed:

```
./five_stage -t traces/sample.tr -c confs/2-wide.conf -v
```

The output should look like:

```
[1: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[1: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[1: IF]    LOAD: (Seq:        1)(Regs:  16,  29, 255)(Addr: 2147450880)(PC: 2097312)
[1: IF]   ITYPE: (Seq:        2)(Regs:  28, 255, 255)(Addr: 4097)(PC: 2097316)
[2: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[2: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[2: IF]   ITYPE: (Seq:        3)(Regs:  28,  28, 255)(Addr: -16384)(PC: 2097320)
[2: IF]   ITYPE: (Seq:        4)(Regs:  17,  29, 255)(Addr: 4)(PC: 2097324)
[3: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[3: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[3: IF]   ITYPE: (Seq:        5)(Regs:   3,  17, 255)(Addr: 4)(PC: 2097328)
[3: IF]   ITYPE: (Seq:        6)(Regs:   2, 255,  16)(Addr: 2)(PC: 2097332)
...
```

The format of one line in the printout is as follows.  For example, let's take this line:

```
[1: IF]    LOAD: (Seq:        1)(Regs:  16,  29, 255)(Addr: 2147450880)(PC: 2097312)
```

* [1: IF] - This says an instruction entered the pipeline at cycle 1 by being fetched at the IF stage.
* LOAD - This says that instruction was a LOAD type instruction.
* (Seq: 1) - This says this instruction was sequence number 1 in the trace file (i.e. first instruction in the file).
* (Regs:  16,  29, 255) - This says registers dst, src1, src2 were 16, 29, 255 respectively.  LOAD instructions do not use the src2 register so it is set to 255 (-1 in signed byte integer).
* (Addr: 2147450880) - This says the LOAD instruction read address 2147450880 in memory.
* (PC: 2097312) - The PC (program counter, or instruction pointer) was 2097312 when the trace was generated.

If the -d (debug) option is given on the commandline, the internal state of pipeline stages is printed at every clock cyle, which can be useful for debugging your simulator.  The -v option is implied by the -d option.  The Makefile script uses the -d option to generate outputs.  You can open the generated output file after building:

```
nano outputs/sample.2-wide.out
```

Or, you can run with the -d option yourself:

```
./five_stage -t traces/sample.tr -c confs/2-wide.conf -d
```

The output should look like:

```
[CYCLE NUMBER: 1]
[1: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[1: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[1: IF]    LOAD: (Seq:        1)(Regs:  16,  29, 255)(Addr: 2147450880)(PC: 2097312)
[1: IF]   ITYPE: (Seq:        2)(Regs:  28, 255, 255)(Addr: 4097)(PC: 2097316)
=================================================================================
                                              Pipeline Stage
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
    NOP: (Seq:        0)(Regs:   0,   0,   0) MEM_ALU
    NOP: (Seq:        0)(Regs:   0,   0,   0) MEM_lwsw
    NOP: (Seq:        0)(Regs:   0,   0,   0) EX_ALU
    NOP: (Seq:        0)(Regs:   0,   0,   0) EX_lwsw
    NOP: (Seq:        0)(Regs:   0,   0,   0) ID
    NOP: (Seq:        0)(Regs:   0,   0,   0) ID
   LOAD: (Seq:        1)(Regs:  16,  29, 255) IF
  ITYPE: (Seq:        2)(Regs:  28, 255, 255) IF
=================================================================================
...
```

This shows the contents of the 2-wide processor at [CYCLE NUMBER: 1] in between
the ========== marks.  Instructions LOAD and ITYPE are at the IF stage because
they have just been fetched.  This is a 2-wide processor so we are able to
process two instructions per cycle.  The top 4 rows show the two instructions
that have just retired (the WB stage) and two instructions that have just been
fetched (the IF stage) in more detail.

```
...
[CYCLE NUMBER: 2]
[2: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[2: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[2: IF]   ITYPE: (Seq:        3)(Regs:  28,  28, 255)(Addr: -16384)(PC: 2097320)
[2: IF]   ITYPE: (Seq:        4)(Regs:  17,  29, 255)(Addr: 4)(PC: 2097324)
=================================================================================
                                              Pipeline Stage
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
    NOP: (Seq:        0)(Regs:   0,   0,   0) MEM_ALU
    NOP: (Seq:        0)(Regs:   0,   0,   0) MEM_lwsw
    NOP: (Seq:        0)(Regs:   0,   0,   0) EX_ALU
    NOP: (Seq:        0)(Regs:   0,   0,   0) EX_lwsw
   LOAD: (Seq:        1)(Regs:  16,  29, 255) ID
  ITYPE: (Seq:        2)(Regs:  28, 255, 255) ID
  ITYPE: (Seq:        3)(Regs:  28,  28, 255) IF
  ITYPE: (Seq:        4)(Regs:  17,  29, 255) IF
=================================================================================
...
```

At [CYCLE NUMBER: 2], the original LOAD and ITYPE instructions are now at the
ID stage.  Two new ITYPE instructions have been fetched at the IF stage.

```
...
[CYCLE NUMBER: 3]
[3: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[3: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[3: IF]   ITYPE: (Seq:        5)(Regs:   3,  17, 255)(Addr: 4)(PC: 2097328)
[3: IF]   ITYPE: (Seq:        6)(Regs:   2, 255,  16)(Addr: 2)(PC: 2097332)
=================================================================================
                                              Pipeline Stage
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
    NOP: (Seq:        0)(Regs:   0,   0,   0) MEM_ALU
    NOP: (Seq:        0)(Regs:   0,   0,   0) MEM_lwsw
  ITYPE: (Seq:        2)(Regs:  28, 255, 255) EX_ALU
   LOAD: (Seq:        1)(Regs:  16,  29, 255) EX_lwsw
  ITYPE: (Seq:        3)(Regs:  28,  28, 255) ID
  ITYPE: (Seq:        4)(Regs:  17,  29, 255) ID
  ITYPE: (Seq:        5)(Regs:   3,  17, 255) IF
  ITYPE: (Seq:        6)(Regs:   2, 255,  16) IF
=================================================================================
...
```

At [CYCLE NUMBER: 3], you will see that the order of LOAD and ITYPE has been
flipped to ITYPE and LOAD.  This is not a change in the ordering in any sense.
For the EX stage, the top row refers to the ALU/Branch EX unit and the bottom
row refers to the lw/sw EX unit.  Since ITYPE belongs to the former and LOAD
belongs to the latter, that's why the ordering changed.  The same applies to
the MEM stage.

```
...
[CYCLE NUMBER: 4]
[4: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[4: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[4: IF]   RTYPE: (Seq:        7)(Regs:   3,   3,   2)(Addr: 0)(PC: 2097336)
=================================================================================
                                              Pipeline Stage
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
  ITYPE: (Seq:        2)(Regs:  28, 255, 255) MEM_ALU
   LOAD: (Seq:        1)(Regs:  16,  29, 255) MEM_lwsw
  ITYPE: (Seq:        3)(Regs:  28,  28, 255) EX_ALU
    NOP: (Seq:        0)(Regs:   0,   0,   0) EX_lwsw
  ITYPE: (Seq:        4)(Regs:  17,  29, 255) ID
  ITYPE: (Seq:        5)(Regs:   3,  17, 255) ID
  ITYPE: (Seq:        6)(Regs:   2, 255,  16) IF
  RTYPE: (Seq:        7)(Regs:   3,   3,   2) IF
=================================================================================
...
```

At [CYCLE NUMBER: 4], we see our first bubble.  This is due to a structural
hazard on the ALU/Branch EX unit.  There were two ITYPE instructions ready to
issue in the ID stage at the previous cycle, but since there is only one
ALU/Branch EX unit, only one could issue.  The lw/sw EX unit is filled with a
NOP bubble.

In this way, you can analyze the output to make sure that various pipeline
hazards are handled properly and also your optimizations work properly.  Again,
there is a set of reference outputs under the outputs_solution/ directory and
diffs in the diffs/ directory that you can also analyze to figure out what you
did wrong.

Now, the simulator can also be run with a 1-wide processor configuration (see
next section [Configuration Files](#configuration-files)).  Let's take a brief
look at the output for that configuration:

```
nano outputs/sample.1-wide.out
```

You will see the following:

```
[CYCLE NUMBER: 1]
[1: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[1: WB]     NOP: (Seq:        0)(Regs:   0,   0,   0)(Addr: 0)(PC: 0)
[1: IF]    LOAD: (Seq:        1)(Regs:  16,  29, 255)(Addr: 2147450880)(PC: 2097312)
=================================================================================
                                              Pipeline Stage
    NOP: (Seq:        0)(Regs:   0,   0,   0) WB
    NOP: (Seq:        0)(Regs:   0,   0,   0) MEM
    NOP: (Seq:        0)(Regs:   0,   0,   0) EX
    NOP: (Seq:        0)(Regs:   0,   0,   0) ID
   LOAD: (Seq:        1)(Regs:  16,  29, 255) IF
=================================================================================
...
```

Since the processor is now 1-wide, there is only each of IF, ID, EX, MEM, and
WB stages per cycle.  The EX stage may be the ALU/Branch EX unit or the lw/sw
EX unit depending upon the instruction type.  Since only one of either can be
active at a time, only one is shown.

## Creating Performance Plots

**UNDER CONSTRUCTION.  DO NOT TRY YET.**

Once you are done implementing the simulator, now you can use it to generate
some performance plots for longer traces of execution.  In order to generate
the plots, do the following:


```
$ make plots
```

The above command will create two files: IPC.pdf and IPC_solution.pdf.  The two
files show the IPCs for the short traces in
/afs/cs.pitt.edu/courses/1541/short_traces for the various processor
configurations under plot_confs/, for your five_stage binary and the
five_stage_solution binary respectively.

If you open IPC_solution.pdf, you will see 8 bars (results of running each of
the 4 short traces on a 1-wide processor and a 2-wide processor).  Each bar is
a histogram with 5 component bars stacked up:

* no optimization: IPC of a processor with no optimizations to avoid hazards
* enableForwarding: Additional IPC gain when enableForwarding is set to true in the configuration file
* branchPredictor: Additional IPC gain when branchPredictor and branchTargetBuffer are set to true in the configuration file
* splitCaches: Additional IPC gain when splitCaches is set to true in the configuration file
* regFileWritePorts=2: Additional IPC gain when regFileWritePorts is increased to 2 (from 1) in the configuration file

Each optimization is turned on incrementally in the order listed above and the IPC gain measured.

Now IPC.pdf (the plot generated from your five_stage) initially will look very
different from IPC_solution.pdf.  It will show no components pertaining to IPC
gains due to optimizations because the optimizations have yet to be
implemented.  Also, the IPCs will be higher because bubbles due to hazards have
not been implemented either.  But once you are done and you pass all the diff
tests, your plot should look identical to the solution plot.

# Configuration Files and Trace Files

## Configuration Files

You can find 4 configuration files under the confs/ directory.  Each will configure a different processor when passed as the -c option to five_stage.  The files are: 1-wide.conf, 1-wide-opt.conf, 2-wide.conf, 2-wide-opt.conf.

* 1-wide.conf : Configuration for a 1-wide processor that fetches and decodes only one instruction per cycle.  Similar to the classic five stage pipeline we discussed extensively in class, except that instructions are issued to different EX units depending on type.
* 1-wide-opt.conf : Configuration for a 1-wide processor with all hazard avoiding hardware optimizations enabled such as data forwarding.
* 2-wide.conf : Configuration for the 2-wide processor described above.
* 2-wide-opt.conf : Configuration for the 2-wide processor with hazard avoiding enabled.

Here is how the 2-wide-opt.conf file looks like:

```
[pipeline]

width=2

[structural hazard]

splitCaches=true
regFileWritePorts=2

[data hazard]

enableForwarding=true

[control hazard]

branchPredictor=true
branchTargetBuffer=true
```

Here is what each of those items mean:

* width=2 : It is a 2-wide processor.
* splitCaches=true : The processor has split caches: one data cache and one instruction cache.  This means all structural hazards between the MEM and IF stages can be resolved without stalling.
* regFileWritePorts=2 : The register file has two write ports.  This means that when two instructions at the WB stage both need to write to the register file, they do not need to stall.
* enableForwarding=true : The processor has data forwarding so that instructions with data dependencies do not need to stall until writeback of the register value.  An exception we learned in class is use-after-load where a register use immediately after a load to that register need to stall one cycle.
* branchPredictor=true : The processor has a branch predictor (essentially a Branch History Table) that can predict the direction of a branch.  100% prediction rate is assumed.  On a predicted taken branch, instructions can start to be fetched after the ID stage when the branch target address is decoded, instead of the EX stage when the branch condition is evaluated.  On a predicted not taken branch, instructions can start to be fetched immediately after the IF stage since the branch target is not needed.
* branchTargetBuffer=true : The processor has a Branch Target Buffer that can predict the branch target as well as the direction.  Again 100% prediction rate is assumed.  Now instruction can start to be fetched immediately after the IF stage regardless of taken or not taken because the target can be predicted at IF even before decoding.

## Trace Files

You can find 8 trace files under the traces/ directory.  Most of them test whether you handled a hazard correctly.

* structural_memory.tr : Structural hazard on the memory read port involving the LOAD MEM stage and another instruction's IF stage.
* structural_regfile.tr : Structural hazard on the register file write port involving an RTYPE and a LOAD instruction both writing to the register file.
* data_load_use.tr : Contains a LOAD followed by an RTYPE (register only) instruction dependent on the load, representing a data hazard.
* data_raw.tr : Another data hazard but this type an RTYPE followed by a dependent RTYPE instruction.  The dependency is RAW (Read After Write).
* data_waw.tr : Again a data hazard with an RTYPE and a LOAD instruction both writing to the same register.  the dependency is WAW (Write After Write).  Note this is different from structural_regfile.tr in that increasing the register write ports would not solve this problem.
* control_taken.tr : Control hazard on a taken BRANCH instruction followed by a couple of other instructions.
* control_non_taken.tr : A not taken BRANCH instruction followed by a couple of other instructions.  Since our processor just keeps on fetching instructions after a branch anyway this is technically not a hazard.  But it's there to make sure you are able to differentiate taken and not taken branches.
* sample.tr : A moderately long trace of instructions (681 instructions) that contain various hazards.

# Your Tasks

## Task 1: Enforcing Stalls and Flushes on Hazards

Our five_stage simulator at its current state does not deal with hazards and hence does not simulate correct execution.  The performance numbers you are seeing are overly optimistic.  Emulate what a Hazard Detection Unit (HDU) would do and inject bubbles into your pipeline where pipeline stalls or flushes are required to handle a hazard.  At this point, assume that all hazard avoidance optimizations are omitted (i.e. we are using the 2-wide.conf or the 1-wide.conf configuration).  Specifically handle the following hazards.

### Structural hazards

I have already handled one structural hazard for you: the one on EX units when two instructions in the ID stage need to be issued to the same EX unit.  The solution was to stall and have the later instruction wait one cycle before issuing.  And these stalls single-handedly brought down the IPC of the 2-wide processor from 2 to 1.2438!

```
grep IPC outputs/sample.2-wide*
```

The output should look like:

```
outputs/sample.2-wide-opt.out:+ IPC (Instructions Per Cycle) : 1.2438
outputs/sample.2-wide.out:+ IPC (Instructions Per Cycle) : 1.2438
```

IPC of 2 is the ideal number for a 2-wide processor and the current implementation inserts no bubbles except for the above hazard.  And that is also why because 2-wide-opt and 2-wide produce the same IPC: because the hazard avoidance optimizations don't have effect when there are no hazards to begin with.

There are two additional structural hazards that I want you to implement:

* The structural hazard on memory read ports: When a LOAD (lw) instruction is at the MEM stage where it reads from memory it is in contention with the IF stage of a later instruction which also reads from memory to fetch instructions.  In this case, the HDU must delay all instruction fetches (both of them) one cycle until the lw MEM stage is over.  Note that a STORE (sw) instruction does not matter because it uses the write port of memory.

* The structural hazard on register file write ports: When a LOAD instruction and an ALU instruction both attempt to write to the register file at the WB stage, there is a structural hazard on the write port, if there is only one.  In this case, the contention should be resolved by having the older instruction go first while the younger remains in the WB stage.

### Data hazards

In our processor, data dependencies through memory don't cause data hazards because all instructions execute in-order and there is only one MEM unit.  Basically, even if there is a LOAD that immediately follows a STORE accessing the same memory location, the value would already be in memory by the time the LOAD gets to the MEM stage.  Hence, all data hazards occur through registers.  Specifically:

* RAW (Read After Write) hazards: This is when an instruction reads a register written by a previous instruction, and there is a possibility that the ordering can be flipped or happen simultaneously.  With no data forwarding, stalls due to RAW hazards happen at the ID stage since that is the stage when registers are read from the register file.  If there is a data hazard, the ID stage would be cancelled and the instruction would wait at the IF stage.  Only when the data hazard is resolved is the instruction allowed to proceed to the ID stage and read registers.  With data forwarding, stall happen at the EX stage since that is the stage where values are forwarded.  If there is a data hazard, the EX stage is canceled and the instruction will wait until the forwarded value is available.

  * Use-after-load hazards: This is a special case of a RAW hazard where the register writing instruction is a LOAD instruction.  This is more problematic because LOAD produces the value at the MEM stage (whereas ALU instructions produce the value at the EX stage), so even with data forwarding, you need a stall as we learned in class.  But without forwarding, it is the same as any RAW hazard since all registers are written back on the WB stage anyway.

* WAW (Write After Write) hazards: This is when an instruction writes to a register written by a previous instruction.  The ordering will never flip on a 1-wide processor since all register writes happen at the WB stage and instructions never overtake each other.  But in a 2-wide processor, when a LOAD instruction and an ALU instruction both attempt to write to the same register at the WB stage, there are no guarantees on who will write first.  Hence stalls are inserted at the WB stage to enforce the correct ordering.  If there is a hazard, the older instruction is allowed to write first to the register file while the younger instruction stalls for one cycle.

* WAR (Write After Read) hazards: This is when an instruction writes to a register read by a previous instruction.  If their orders are flipped, then this too can produce wrong outcomes, just like RAW or WAW hazards.  However, for a WAR hazard, our in-order processor design guarantees the correct ordering because writes happen at the WB stage and reads happen at the ID stage.  There is no situation where the WB stage of a younger instruction overtakes the ID stage of an older instruction.  So we do not need to worry about this particular hazard.  As an aside, if our processor had an out-of-order design where instructions are dynamically scheduled to execute out of order, WAR hazards would have been a cause for concern.

* RAR (Read After Read) hazards?  No such thing exists.  Two reads to a register can happen in any order without changing the outcome. :)

### Control hazards

A control hazard occurs because on a conditional branch instruction, there is a delay between the fetch of the instruction and when the condition is resolved in the EX stage.  Meanwhile the processor keeps on fetching subsequent instructions using PC + 4.  At the very next instruction the branch instruction is at the ID stage so the processor doesn't even yet know this is a branch.  At the next, next instruction, the branch is at the EX stage so the processor knows that it is a branch instruction by now but still doesn't know the direction the branch will go.  Now if the branch turns out to be a not taken branch then there is no control hazard.  The instructions you fetched meanwhile turned out to be the correct instructions!  But if the branch turns out to be taken, the processor needs to flush the pipeline of those "meanwhile" instructions and start from the correct path.

In terms of your simulator, this means on a taken branch, inserting bubbles into the pipeline until the branch EX stage is over.  How do you know if it is a taken branch?  Fortunately, the generated trace entry already contains the Addr item which is the target address that the branch jumped (or did not jump) to.  So if it is a taken branch, emulate the pipeline flush by inserting bubbles at the IF stage.

## Task 2: Enabling Optimizations on the Hazards

Enable all the hazard avoidance optimizations that were described in the [Configuration Files](#configuration-files) section.  Be careful that while the optimizations will reduce hazards drastically, there are some hazards that remain even after the optimizations.

## Source Code

Each trace file is a sequence of dynamic trace items, where each trace item represents one instruction executed in the program that has been traced. After five_stage.c  reads a trace item, it stores it in a structure:

```
struct instruction {
        uint8_t type;                 	// holds the op-code - see below
        uint8_t sReg_a;              	// 1st operand
        uint8_t sReg_b;              	// 2nd operand
        uint8_t dReg;                	// dest. operand
        uint32_t PC;                 	// program counter
        uint32_t Addr;               	// mem. address
};
```
where:
```
enum opcode_type {
        ti_NOP = 0,
        ti_RTYPE,
        ti_ITYPE,
        ti_LOAD,
        ti_STORE,
        ti_BRANCH,
        ti_JTYPE,
        ti_SPECIAL,
        ti_JRTYPE
};
```

The “PC” (program counter) field is the address of the instruction itself. The “type” of an instruction provides the key information about the instruction. A detailed list of instructions is given below:

```
NOP - it's a no-op. No further information is provided.
RTYPE - An R-type instruction.
        sReg_a: first register operand (register name)
        sReg_b: second register operand (register name)
        dReg: destination register name
        PC: program counter of this instruction
        Addr: not used
ITYPE - An I-type instruction that is not LOAD, STORE, or BRANCH.
        sReg_a: first register operand (register name)
        sReg_b: not used
        dReg: destination register name
        PC: program counter of this instruction
        Addr: immediate value
LOAD - a load instruction (memory access)
        sReg_a: first register operand (register name)
        sReg_b: not used
        dReg: destination register name
        PC: program counter of this instruction
        Addr: memory address
STORE - a store instruction (memory access)
        sReg_a: first register operand (register name)
        sReg_b: second register operand (register name)
        dReg: not used
        PC: program counter of this instruction
        Addr: memory address
BRANCH - a branch instruction
        sReg_a: first register operand (register name)
        sReg_b: second register operand (register name)
        dReg: not used
        PC: program counter of this instruction
        Addr: target address (the actual address of the next executed instruction)
JTYPE - a jump instruction
        sReg_a: not used
        sReg_b: not used
        dReg: not used
        PC: program counter of this instruction
        Addr: target address (the actual address of the next executed instruction)
SPECIAL - it's a special system call instruction
        For now, ignore other fields of this instruction.
JRTYPE - a jump register instruction (used for "return" in functions)
        sReg_a: source register (that keeps the target address)
        sReg_b: not used
        dReg: not used
        PC: program counter of this instruction
        Addr: target address (the actual address of the next executed instruction)
```

## Submission

Each pairwise group will submit the exercise *once* to GradeScope, by *one
member* of the group.  The submitting member will press the "View or edit
group" link at the top-right corner of the assignment page after submission to
add his/her partner.  That way, both of you will get a grade.  This applies to
both the Project 1 Source Code and Project 1 Retrospective submissions
explained below.

You will do two submissions for this deliverable.

1. **(90 points)** Project 1 Soure Code (Due Mar 16, 2022 4:30 PM)

   The easiest way to submit the source code is by submitting your GitHub
repository.  Add your partner as a collaborator so both of you have access.
Make sure you keep the repository **PRIVATE** so that nobody else can access
your repository.  

   Once you are done modifying code, don't forget to commit and push your
changes to the github repository.  Before you commit your changes, please do
'make distclean' so that you don't commit your large debug output files with
your source code.  When you are done, submit your GitHub repository to
GradeScope at the "Project 1 Soure Code" link.  Once you submit, GradeScope
will run the autograder to grade you and give you instant feedback.  If you get
deductions, fix your code based on the feedback and resubmit.  Repeat until you
don't get deductions.  You will get deductions based on the number of failed
diffs (-5 per failed diff).  This part of the submission is worth 90 points.
You start out with 18 failed diffs (out of 32) with no modifications to code,
so you start out with 90 - 18 * 5 = 0 points.
    
   The alternative way to submit your source code is to directly upload the
files.  All you have to do is drag and drop source files that you have modified
on to the Upload window and press the Upload button.  The default files in this
folder will be used for any files that you don't upload.  While this method in
the short run is simpler than the GitHub method, you will find that using a
source repository like GitHub is beneficial in the long run.
    
1. **(20 points)** Project 1 Retrospective (Due TBD)

   **NOT AVAILABLE YET**
   
   Click on the GradeScope "Project 1 Retrospective" link and answer the
questions based on [Task 3: Analyzing Performance
Plots](#task-3-analyzing-performance-plots).  Note that this submission is
worth 20, not 10, points.  That is because 10 points are **extra credit**.  10
points worth of questions are either multiple choice or fill-in-the-blank
questions which are easily answerable by simply reading the plot.  10 points
worth of questions (the extra credit) are short answer questions that ask you
to explain and make conjectures about the experimental results, and require
some additional thinking.
    
   Just like other scientists, computer architects try to draw conclusions
from experimental data from their simulators.  Some conclusions are strongly
supported by the data and other conclusions less so.  In a proper scientific
publication, architects publish both the raw data and the conclusions they
derived from it.  It is up to the reader to judge the strength of each
conclusion based on the data provided.  By doing the extra credit, you will
practice drawing conclusions from data, just like architects do every day.
Since this is extra credit, I will expect you to write a reasonable conclusion
--- something that is not supported by the data and your knowledge of the
simulator will not get you any points.
    
# Resources

* Windows SSH Terminal Client: [Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
* GitHub GUI Client: [GitHub Desktop](https://desktop.github.com/)
* File Transfer Client: [FileZilla](https://filezilla-project.org/download.php?type=client)
* Linux command line tutorial: [The Linux Command Line](http://linuxcommand.org/lc3_learning_the_shell.php)
* Valgrind tutorial: [Valgrind tutorial](https://valgrind.org/docs/manual/QuickStart.html)
* GDB tutorial: [GDB tutorial](https://sourceware.org/gdb/current/onlinedocs/gdb/)
* DDD tutorial: [DDD tutorial](https://www.gnu.org/software/ddd/manual/html_mono/ddd.html)

## GitHub Primer

Accepting the GitHub Classroom assignment should have created a GitHub
repository for your group.  Git is one of the most popular source versioning
and collaboration tools used in industry and GitHub is a major provider of that
service.  By using Git, your project doesn't evaporate if you spill coffee on
your laptop.  Also, it allows you and your partner to work on the same source
code without stepping on each other's toes by maintaining a central repository.  


1. You need a clone of the Remote Repository on GitHub on your machine to start
   working on it.  Here is a short tutorial on how to clone repositories:

   https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository

2. Whenever you make improvements to your source code, frequently "Commit" and
   "Push" those changes to GitHub so that your new changes are versioned.
Committing your changes will transfer the changes from the source code that you
are working on to the Local Repository creating a new version.  Pushing your
changes will upload new versions in your Local Repository to the central Remote
Repository at GitHub.com.  So only after you Push will the changes be available
to your collaborator to Pull (or yourself from a different machine).
Committing and Pushing frequently ensures that your changes are versioned and
backed up, as well as allowing your collaborators to access your changes and
keep up-to-date.  Here are a couple of tutorials on how to Commit and Push:

   https://github.com/git-guides/git-commit

   https://github.com/git-guides/git-push

3. Whenever there are updates to the Remote Repository (somebody did a "Push"
   to the repository), you need to "Pull" the changes to your Local Repository.
Before you start modifying the code make it a habit to Pull to make sure your
Local Repository in sync with the Remote Repository at GitHub.com.  Also,
whenever you Push, notify your group members so that they can Pull.  Here is a short tutorial on how to Pull:

   https://github.com/git-guides/git-pull

## Debugging C

You can use all the tools that you learned in CS 449 to debug your simulator,
including GDB (the GNU Debugger) and Valgrind.  Here are couple of tutorials to
jog your memory:

* Valgrind tutorial: [Valgrind tutorial](https://valgrind.org/docs/manual/QuickStart.html)
* GDB tutorial: [GDB tutorial](https://sourceware.org/gdb/current/onlinedocs/gdb/)

If you want a graphical interface to GDB there is DDD (Data Display Debugger):

* DDD tutorial: [DDD tutorial](https://www.gnu.org/software/ddd/manual/html_mono/ddd.html)

GDB, Valgrind, and DDD are all available on thoth.cs.pitt.edu, but for you to
be able to access the DDD GUI over SSH, you will need X11 forwarding enabled.

## Using DDD through remote X11 forwarding

This is how you can use the DDD installation on thoth remotely.

### For Windows 10

Enabling X11 forwarding on Windows 10 is a bit complicated since you have to do
deal with Windows firewall.  Here is an easy to follow guide written by another
Computer Architecture professor from the University of Illinois (which also
happens to be my alma mater :).

* If you have WSL1 installed: https://cs233.github.io/oyom_wsl1_setup.html
* If you have WSL2 installed: https://cs233.github.io/oyom_wsl2_setup.html

### For MacOS

Enabling X11 forwarding on Mac systems is more straightforward.  All you have to do install XQuartz and launch it:

* XQuartz download URL: https://www.xquartz.org/

### For Linux

For most Linux systems, X11 forwarding should be built-in with the X windows
system, so no additional installations are needed.

### Establishing SSH connection with X11 forwarding

After enabling X11 forwarding, you have to specify X11 forwarding on your SSH
connection.  When you connect to thoth, use the following commandline:

```
ssh -XC USERNAME@thoth.cs.pitt.edu
```

The -X option enables X11 forwarding and the -C option enables packet
compression on your SSH connection so you can minimize the bandwidth consumed
by X11 forwarding.

If all goes well, after you log on to thoth, your $DISPLAY variable should be set up automatically.

```
wahn@thoth:~$ echo $DISPLAY
localhost:10.0
```

If your $DISPLAY is not set up, then that means that something went wrong.
Either VcXsrv (for Windows) or XQuartz (for Mac) was not set up or something
else.  Note that you should not force set $DISPLAY on your .bashrc file.  That
is not going to achieve anything.  $DISPLAY should be automatically set by SSH.

Once the above is confirmed, you can start using any GUI app on thoth.  Now try
launching DDD:

```
ddd
```

The DDD GUI should pop up on your machine momentarily.

## Using DDD by installing on local machine

This is how you can install DDD on your local machine.

### For Windows 10 WSL or Linux

These are the packages required to compile the project and run DDD, using the
apt package manager.  Please do the following on your linux shell:

```
sudo apt install gcc
```
```
sudo apt install g++
```
```
sudo apt install make
```
```
sudo apt install libglib2.0-dev
```
```
sudo apt install libcairo2-dev
```
```
sudo apt install libpango1.0-dev
```
```
sudo apt install gdb
```
```
sudo apt install x11-apps
```
```
sudo apt install ddd
```

### For Mac

Unfortunately, I gdb (and by extension ddd) does not run reliably on MacOS.
It bas been a long standing problem but has still not been solved:
https://sourceware.org/bugzilla/show_bug.cgi?id=24069

At least, you can still compile and test the program on your local machine.
If you want to use gdb or ddd, you will have to do it remotely on thoth.  At
below are the steps to install the necessary packages.

First you will have to install the XCode commandline developer tools, which
includes gcc and g++:
```
xcode-select --install
```

Then use Homebrew to install all required libraries:
```
brew install glib
```

You will also have to add some modifications to the Makefile:

Change the COPT line from:
```
COPT = -g -Wall -I/usr/include/glib-2.0/ -I/usr/lib/x86_64-linux-gnu/glib-2.0/include/
```
to:
```
COPT = -g -Wall -I/usr/include/glib-2.0/ -I/usr/lib/x86_64-linux-gnu/glib-2.0/include/ -I/usr/local/opt/glib/include/glib-2.0/ -I/usr/local/opt/glib/lib/glib-2.0/include/
```

Also change the LOPT line from:
```
LOPT = -lglib-2.0
```
to
```
LOPT = -lglib-2.0 -L/usr/local/opt/glib/lib/
```

Lastly, replace all mentions of "five_stage_solution" to
"five_stage_solution.mac".  The five_stage_solution binary was compiled on
thoth (a Linux machine) which cannot run on a Mac.

## DDD hang issue solution

There is a long standing bug with DDD where it can hang indefinitely when
launched for the second time.  It is because of some configuration settings
that get auto-generated the first time it is launched.  The solution is to edit
the ~/.ddd/init file to change the following line:

```
set extended-prompt not set\n\
```

To this line:

```
set extended-prompt (gdb) \n\
```

Reference: https://savannah.gnu.org/bugs/index.php
