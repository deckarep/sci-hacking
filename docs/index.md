## SCI Hacking

The purpose of this *small* blog is two-fold: First and foremost, this is an exercise in the study of a classic virtual machine implementation implemented by Sierra Online during the golden era of adventure gaming between the late 80s to mid 90s. Secondly this is to understand a little bit of computing history developed by a gaming company which was largely responsible for motivating me to one day enter the field of software development.

The company Sierra Online founded by Ken and Roberta Williams had the foresight to knowing that they wanted their games to run on as many relevant personal computing architectures during this revolution in gaming. Rather than write then re-write many of the their games to support each platform, Sierra invested heavily in a virtual machine architecture modeled after a 16-bit P-Machine. This computing model that they developed had its own micro-kernel for things like drawing routines, file IO, sound manipulation, etc. It also also had over 100 opcodes and was built on the concept of a stack-machine based VM. This VM also had support for strings, pointers, asset management and many *modern* things you'd find in an implementation today.

On top of this VM architecture, Sierra built a full-blown programming language that was very much modeled after Scheme and Small-Talk. This language supported procedures, object-oriented concepts, local and global variables and was able to be fully compiled. It did not however support garbage-collection and required the programmers to implement their own resource cleanup to keep memory usage bounded.

### Resources

Here's a list of some important resources for getting started with hacking the SCI virtual machine.

### The SCI Virtual Machine
* [ScummVM SCI Engine](https://github.com/scummvm/scummvm/tree/master/engines/sci)
* [SCI P-Machine instruction-set](https://wiki.scummvm.org/index.php/SCI/Specifications/SCI_virtual_machine/The_Sierra_PMachine)
* [SCI Decompilation Archive](https://github.com/EricOakford/SCI-Decompilation-Archive)
* [Original SCI-16 Engine](https://github.com/OmerMor/SCI16)
* [Original SCI-32 Engine](https://github.com/OmerMor/SCI32)
* [SCI Programming Forum](http://sciprogramming.com/)

### Reverse Engineering Tools
* [SCI Companion Website](http://scicompanion.com/)
* [SCI1.1 Class Library](http://scicompanion.com/Documentation/classlibrary.html)
* [SCI Companion Source (not maintained)](https://github.com/icefallgames/SCICompanion)
* [SCI Companion (maintained)](https://github.com/Kawa-oneechan/SCICompanion)

### Blogs
* [Show Your Working: Patching a 25yr old Sierra game](https://moral.net.au/writing/2017/09/23/sierra_bug/)
* [Kawa Blog - SCI development](http://helmet.kafuka.org/logopending/)

### Going Deeper
* [Kawa misc tools](http://helmet.kafuka.org/sci/)

### ScummVM SCI Debugger

The following debugger commands are *specific* to the debugger console built into the `ScummVM` engine.

**Tip**: Starting `ScummVM` from a `terminal` session allows you to view some important output that is logged to `STDOUT` when executing some debug commands. For serious debugging, *you'll want to do this* to get a richer experience when debugging.

**Note**: This list of commands is certainly not exhaustive and some commands only work on certain version of the SCI engine. Typing `help` will show all the available commands but if you get stuck [check the source code](https://github.com/scummvm/scummvm/blob/master/engines/sci/console.cpp) for a further understanding of how the debug commands work.

#### Starting ScummVM for debugging
```sh
# Navigate to the directory of where ScummVM app is installed:
cd ScummVM.app/Contents/MacOS

# Type this to run it from the terminal:
./scummvm
```

#### Open debugger console
MacOS: `[OPTION] + [CTRL] + [SHIFT] + D`

#### Common commands to know
```sh
# Dismiss debugger console (press ESC) or:
) exit
) go

# Get help by typing this into the console:
) help

# Get help on how memory addressing works in the debugger:
) addresses

# Get version of game:
) version

# Restart the game:
) restart_game

# Quit the game:
) quit

# Change rooms:
) room <room-num>
) room 440

# Execute a selector to read the memory:
) send <obj> <selector>
) send theBadge x

# Execute a selector to update memory:
) send <obj> <selector> [arg1, arg2, ...]
) send theBadge x 150

# Inspect scripts currently loaded into memory:
) scro *

# View an object properties and methods:
) vo <obj>
) vo theBadge

```

#### Probing memory

SCI games use a 2 component 16bit addressing system that comprises of two parts depending on what they hold.

* When used as a pointer both components are used like so:
  * `[0032]:[2cf4]`
  * The first component is the segment id and the second component is some absolute or relative offset into memory.
* When used as just pure data such as a y coordinate they look like so:
  * `[0000]:[0005]`
  * The first component is just all zeroes since it's not really a pointer and the second component is some value.

```sh
# Inspect the registers
) registers

# Inspect the stack
) stack <depth>
) stack 10

# List variables (global, local, temp, param)
) vmvarlist or vl

# Display or change vars (global, local, temp, param)
) vv <type> <varnum> [<value>]
) vv g 0
```

#### Stepping through code
```sh
# View current breakpoints
) bl

# Set a break point
) bpx <obj>::<method-selector>
) bpx theBadge::init

# Once triggered step through the code using:
) s

# Delete breakpoint
) bp_del <name>
) bp_del * # deletes all

# Show the method stack-trace (also called backtrace): 
) bt
```

#### Disassembling methods
```sh
# Dump disassembly instructions on an object method to the terminal via STDOUT:
) disasm <obj> <method-selector> bcc
) disasm gammie doit bcc
```

### Bytecode VM hacks

The VM doesn't have the concept of a NOP instruction (which is an instruction that simply does nothing). Therefore these two methods are recommended to accomplish a similar thing for a patch.

```sh
# Use jump instruction to skip over instructions effectively making them no-ops.
jmp <relpos>

# Another method is to empty the accumulator to cause instructions to operate against nothing or a NULL pointer.
ldi 0000
```

### Example Hacks

#### Leisure Suit Larry 6 Hack - Cav Easter Egg Exposed

How this patch works: Since the logic for the game is located in file `440.SCR` and this resource is *not compressed*, the bytecode instruction set is available for hex-editing.

In total *3 bytes* must be changed. Since two selectors are enqueued to be executed for the asset in question, we effectively need to prevent the `hide` one from running. We accomplish this by changing the instructions to use a `jmp 1` instead of `pushi 66` where `0x66` is the selector for `hide` replacing the two bytes responsible for enqueueing `hide` on the stack as the second selector to be executed. (The other is `init` which must still run.)

We're not quite done...in the original code since the `stack` is setup to handle 2 selector calls this means that `send 0x08` should be `send 0x04` instead because the stack is effectively halved since the `hide` selector was not pushed and we need to make sure that the `send` command accounts for only *4 bytes* on the stack vs 8 bytes.

```sh
# lsl6 dos (non-hires) - (room 440)
# ) disasm closeUpInset init bcc

# Patch is applied to file: 440.SCR

# Before
0036:1a5e: 0x39, 0x6e,                          // pushi        6e              ; 110, 'n', init
0036:1a60: 0x76,                                // push0
0036:1a61: 0x39, 0x66,                          // pushi        66              ; 102, 'f', hide
0036:1a63: 0x76,                                // push0
0036:1a64: 0x72, 0xba, 0x04,                    // lofsa        tits[2c05]
0036:1a67: 0x4a, 0x08,                          // send         08

# After
0036:1a5e: 0x39, 0x6e,                          // pushi        6e              ; 110, 'n', init
0036:1a60: 0x76,                                // push0
0036:1a61: 0x33, 0x01,                          // jmp          01              ; jump +1
0036:1a63: 0x76,                                // push0                        ; <SKIPPED FROM JUMP>
0036:1a64: 0x72, 0xba, 0x04,                    // lofsa        tits[2c05]
0036:1a67: 0x4a, 0x04,                          // send         04              ; 4 instead of 8 since we're invoking one selector: (init) instead of two selectors: (init & hide).
```



### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/deckarep/sci-hacking/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
