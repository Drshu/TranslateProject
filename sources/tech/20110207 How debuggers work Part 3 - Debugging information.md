<header class="page-header" style="padding-bottom: 9px; margin-top: 40px; margin-bottom: 10px; font-family: &quot;Droid Sans&quot;, sans-serif; font-size: 16px; text-align: start; background-color: rgb(255, 255, 255);">[How debuggers work: Part 3 - Debugging information][25]
============================================================</header>

<footer class="post-info"> <time>February 07, 2011 at 19:02</time> Tags [Articles][8] , [Debuggers][9] , [Programming][10]</footer>

This is the third part in a series of articles on how debuggers work. Make sure you read [the first][26] and [the second][27] parts before this one.

### In this part

I'm going to explain how the debugger figures out where to find the C functions and variables in the machine code it wades through, and the data it uses to map between C source code lines and machine language words.

### Debugging information

Modern compilers do a pretty good job converting your high-level code, with its nicely indented and nested control structures and arbitrarily typed variables into a big pile of bits called machine code, the sole purpose of which is to run as fast as possible on the target CPU. Most lines of C get converted into several machine code instructions. Variables are shoved all over the place - into the stack, into registers, or completely optimized away. Structures and objects don't even  _exist_  in the resulting code - they're merely an abstraction that gets translated to hard-coded offsets into memory buffers.

So how does a debugger know where to stop when you ask it to break at the entry to some function? How does it manage to find what to show you when you ask it for the value of a variable? The answer is - debugging information.

Debugging information is generated by the compiler together with the machine code. It is a representation of the relationship between the executable program and the original source code. This information is encoded into a pre-defined format and stored alongside the machine code. Many such formats were invented over the years for different platforms and executable files. Since the aim of this article isn't to survey the history of these formats, but rather to show how they work, we'll have to settle on something. This something is going to be DWARF, which is almost ubiquitously used today as the debugging information format for ELF executables on Linux and other Unix-y platforms.

### The DWARF in the ELF

 ![](http://eli.thegreenplace.net/images/2011/02/dwarf_logo.gif) 

According to [its Wikipedia page][17], DWARF was designed alongside ELF, although it can in theory be embedded in other object file formats as well [[1]][18].

DWARF is a complex format, building on many years of experience with previous formats for various architectures and operating systems. It has to be complex, since it solves a very tricky problem - presenting debugging information from any high-level language to debuggers, providing support for arbitrary platforms and ABIs. It would take much more than this humble article to explain it fully, and to be honest I don't understand all its dark corners well enough to engage in such an endeavor anyway [[2]][19]. In this article I will take a more hands-on approach, showing just enough of DWARF to explain how debugging information works in practical terms.

### Debug sections in ELF files

First let's take a glimpse of where the DWARF info is placed inside ELF files. ELF defines arbitrary sections that may exist in each object file. A  _section header table_  defines which sections exist and their names. Different tools treat various sections in special ways - for example the linker is looking for some sections, the debugger for others.

We'll be using an executable built from this C source for our experiments in this article, compiled into tracedprog2:

```
#include <stdio.h>

void do_stuff(int my_arg)
{
    int my_local = my_arg + 2;
    int i;

    for (i = 0; i < my_local; ++i)
        printf("i = %d\n", i);
}

int main()
{
    do_stuff(2);
    return 0;
}
```

Dumping the section headers from the ELF executable using objdump -h we'll notice several sections with names beginning with .debug_ - these are the DWARF debugging sections:

```
26 .debug_aranges 00000020  00000000  00000000  00001037
                 CONTENTS, READONLY, DEBUGGING
27 .debug_pubnames 00000028  00000000  00000000  00001057
                 CONTENTS, READONLY, DEBUGGING
28 .debug_info   000000cc  00000000  00000000  0000107f
                 CONTENTS, READONLY, DEBUGGING
29 .debug_abbrev 0000008a  00000000  00000000  0000114b
                 CONTENTS, READONLY, DEBUGGING
30 .debug_line   0000006b  00000000  00000000  000011d5
                 CONTENTS, READONLY, DEBUGGING
31 .debug_frame  00000044  00000000  00000000  00001240
                 CONTENTS, READONLY, DEBUGGING
32 .debug_str    000000ae  00000000  00000000  00001284
                 CONTENTS, READONLY, DEBUGGING
33 .debug_loc    00000058  00000000  00000000  00001332
                 CONTENTS, READONLY, DEBUGGING
```

The first number seen for each section here is its size, and the last is the offset where it begins in the ELF file. The debugger uses this information to read the section from the executable.

Now let's see a few practical examples of finding useful debug information in DWARF.

### Finding functions

One of the most basic things we want to do when debugging is placing breakpoints at some function, expecting the debugger to break right at its entrance. To be able to perform this feat, the debugger must have some mapping between a function name in the high-level code and the address in the machine code where the instructions for this function begin.

This information can be obtained from DWARF by looking at the .debug_info section. Before we go further, a bit of background. The basic descriptive entity in DWARF is called the Debugging Information Entry (DIE). Each DIE has a tag - its type, and a set of attributes. DIEs are interlinked via sibling and child links, and values of attributes can point at other DIEs.

Let's run:

```
objdump --dwarf=info tracedprog2
```

The output is quite long, and for this example we'll just focus on these lines [[3]][20]:

```
<1><71>: Abbrev Number: 5 (DW_TAG_subprogram)
    <72>   DW_AT_external    : 1
    <73>   DW_AT_name        : (...): do_stuff
    <77>   DW_AT_decl_file   : 1
    <78>   DW_AT_decl_line   : 4
    <79>   DW_AT_prototyped  : 1
    <7a>   DW_AT_low_pc      : 0x8048604
    <7e>   DW_AT_high_pc     : 0x804863e
    <82>   DW_AT_frame_base  : 0x0      (location list)
    <86>   DW_AT_sibling     : <0xb3>

<1><b3>: Abbrev Number: 9 (DW_TAG_subprogram)
    <b4>   DW_AT_external    : 1
    <b5>   DW_AT_name        : (...): main
    <b9>   DW_AT_decl_file   : 1
    <ba>   DW_AT_decl_line   : 14
    <bb>   DW_AT_type        : <0x4b>
    <bf>   DW_AT_low_pc      : 0x804863e
    <c3>   DW_AT_high_pc     : 0x804865a
    <c7>   DW_AT_frame_base  : 0x2c     (location list)
```

There are two entries (DIEs) tagged DW_TAG_subprogram, which is a function in DWARF's jargon. Note that there's an entry for do_stuff and an entry for main. There are several interesting attributes, but the one that interests us here is DW_AT_low_pc. This is the program-counter (EIP in x86) value for the beginning of the function. Note that it's 0x8048604 for do_stuff. Now let's see what this address is in the disassembly of the executable by running objdump -d:

```
08048604 <do_stuff>:
 8048604:       55           push   ebp
 8048605:       89 e5        mov    ebp,esp
 8048607:       83 ec 28     sub    esp,0x28
 804860a:       8b 45 08     mov    eax,DWORD PTR [ebp+0x8]
 804860d:       83 c0 02     add    eax,0x2
 8048610:       89 45 f4     mov    DWORD PTR [ebp-0xc],eax
 8048613:       c7 45 (...)  mov    DWORD PTR [ebp-0x10],0x0
 804861a:       eb 18        jmp    8048634 <do_stuff+0x30>
 804861c:       b8 20 (...)  mov    eax,0x8048720
 8048621:       8b 55 f0     mov    edx,DWORD PTR [ebp-0x10]
 8048624:       89 54 24 04  mov    DWORD PTR [esp+0x4],edx
 8048628:       89 04 24     mov    DWORD PTR [esp],eax
 804862b:       e8 04 (...)  call   8048534 <printf@plt>
 8048630:       83 45 f0 01  add    DWORD PTR [ebp-0x10],0x1
 8048634:       8b 45 f0     mov    eax,DWORD PTR [ebp-0x10]
 8048637:       3b 45 f4     cmp    eax,DWORD PTR [ebp-0xc]
 804863a:       7c e0        jl     804861c <do_stuff+0x18>
 804863c:       c9           leave
 804863d:       c3           ret
```

Indeed, 0x8048604 is the beginning of do_stuff, so the debugger can have a mapping between functions and their locations in the executable.

### Finding variables

Suppose that we've indeed stopped at a breakpoint inside do_stuff. We want to ask the debugger to show us the value of the my_local variable. How does it know where to find it? Turns out this is much trickier than finding functions. Variables can be located in global storage, on the stack, and even in registers. Additionally, variables with the same name can have different values in different lexical scopes. The debugging information has to be able to reflect all these variations, and indeed DWARF does.

I won't cover all the possibilities, but as an example I'll demonstrate how the debugger can find my_local in do_stuff. Let's start at .debug_info and look at the entry for do_stuff again, this time also looking at a couple of its sub-entries:

```
<1><71>: Abbrev Number: 5 (DW_TAG_subprogram)
    <72>   DW_AT_external    : 1
    <73>   DW_AT_name        : (...): do_stuff
    <77>   DW_AT_decl_file   : 1
    <78>   DW_AT_decl_line   : 4
    <79>   DW_AT_prototyped  : 1
    <7a>   DW_AT_low_pc      : 0x8048604
    <7e>   DW_AT_high_pc     : 0x804863e
    <82>   DW_AT_frame_base  : 0x0      (location list)
    <86>   DW_AT_sibling     : <0xb3>
 <2><8a>: Abbrev Number: 6 (DW_TAG_formal_parameter)
    <8b>   DW_AT_name        : (...): my_arg
    <8f>   DW_AT_decl_file   : 1
    <90>   DW_AT_decl_line   : 4
    <91>   DW_AT_type        : <0x4b>
    <95>   DW_AT_location    : (...)       (DW_OP_fbreg: 0)
 <2><98>: Abbrev Number: 7 (DW_TAG_variable)
    <99>   DW_AT_name        : (...): my_local
    <9d>   DW_AT_decl_file   : 1
    <9e>   DW_AT_decl_line   : 6
    <9f>   DW_AT_type        : <0x4b>
    <a3>   DW_AT_location    : (...)      (DW_OP_fbreg: -20)
<2><a6>: Abbrev Number: 8 (DW_TAG_variable)
    <a7>   DW_AT_name        : i
    <a9>   DW_AT_decl_file   : 1
    <aa>   DW_AT_decl_line   : 7
    <ab>   DW_AT_type        : <0x4b>
    <af>   DW_AT_location    : (...)      (DW_OP_fbreg: -24)
```

Note the first number inside the angle brackets in each entry. This is the nesting level - in this example entries with <2> are children of the entry with <1>. So we know that the variable my_local (marked by the DW_TAG_variable tag) is a child of the do_stuff function. The debugger is also interested in a variable's type to be able to display it correctly. In the case of my_local the type points to another DIE - <0x4b>. If we look it up in the output of objdump we'll see it's a signed 4-byte integer.

To actually locate the variable in the memory image of the executing process, the debugger will look at the DW_AT_location attribute. For my_local it says DW_OP_fbreg: -20. This means that the variable is stored at offset -20 from the DW_AT_frame_base attribute of its containing function - which is the base of the frame for the function.

The DW_AT_frame_base attribute of do_stuff has the value 0x0 (location list), which means that this value actually has to be looked up in the location list section. Let's look at it:

```
$ objdump --dwarf=loc tracedprog2

tracedprog2:     file format elf32-i386

Contents of the .debug_loc section:

    Offset   Begin    End      Expression
    00000000 08048604 08048605 (DW_OP_breg4: 4 )
    00000000 08048605 08048607 (DW_OP_breg4: 8 )
    00000000 08048607 0804863e (DW_OP_breg5: 8 )
    00000000 <End of list>
    0000002c 0804863e 0804863f (DW_OP_breg4: 4 )
    0000002c 0804863f 08048641 (DW_OP_breg4: 8 )
    0000002c 08048641 0804865a (DW_OP_breg5: 8 )
    0000002c <End of list>
```

The location information we're interested in is the first one [[4]][21]. For each address where the debugger may be, it specifies the current frame base from which offsets to variables are to be computed as an offset from a register. For x86, bpreg4 refers to esp and bpreg5 refers to ebp.

It's educational to look at the first several instructions of do_stuff again:

```
08048604 <do_stuff>:
 8048604:       55          push   ebp
 8048605:       89 e5       mov    ebp,esp
 8048607:       83 ec 28    sub    esp,0x28
 804860a:       8b 45 08    mov    eax,DWORD PTR [ebp+0x8]
 804860d:       83 c0 02    add    eax,0x2
 8048610:       89 45 f4    mov    DWORD PTR [ebp-0xc],eax
```

Note that ebp becomes relevant only after the second instruction is executed, and indeed for the first two addresses the base is computed from esp in the location information listed above. Once ebp is valid, it's convenient to compute offsets relative to it because it stays constant while esp keeps moving with data being pushed and popped from the stack.

So where does it leave us with my_local? We're only really interested in its value after the instruction at 0x8048610 (where its value is placed in memory after being computed in eax), so the debugger will be using the DW_OP_breg5: 8 frame base to find it. Now it's time to rewind a little and recall that the DW_AT_location attribute for my_local says DW_OP_fbreg: -20. Let's do the math: -20 from the frame base, which is ebp + 8. We get ebp - 12. Now look at the disassembly again and note where the data is moved from eax - indeed, ebp - 12 is where my_local is stored.

### Looking up line numbers

When we talked about finding functions in the debugging information, I was cheating a little. When we debug C source code and put a breakpoint in a function, we're usually not interested in the first  _machine code_  instruction [[5]][22]. What we're  _really_  interested in is the first  _C code_  line of the function.

This is why DWARF encodes a full mapping between lines in the C source code and machine code addresses in the executable. This information is contained in the .debug_line section and can be extracted in a readable form as follows:

```
$ objdump --dwarf=decodedline tracedprog2

tracedprog2:     file format elf32-i386

Decoded dump of debug contents of section .debug_line:

CU: /home/eliben/eli/eliben-code/debugger/tracedprog2.c:
File name           Line number    Starting address
tracedprog2.c                5           0x8048604
tracedprog2.c                6           0x804860a
tracedprog2.c                9           0x8048613
tracedprog2.c               10           0x804861c
tracedprog2.c                9           0x8048630
tracedprog2.c               11           0x804863c
tracedprog2.c               15           0x804863e
tracedprog2.c               16           0x8048647
tracedprog2.c               17           0x8048653
tracedprog2.c               18           0x8048658
```

It shouldn't be hard to see the correspondence between this information, the C source code and the disassembly dump. Line number 5 points at the entry point to do_stuff - 0x8040604. The next line, 6, is where the debugger should really stop when asked to break in do_stuff, and it points at 0x804860a which is just past the prologue of the function. This line information easily allows bi-directional mapping between lines and addresses:

*   When asked to place a breakpoint at a certain line, the debugger will use it to find which address it should put its trap on (remember our friend int 3 from the previous article?)
*   When an instruction causes a segmentation fault, the debugger will use it to find the source code line on which it happened.

### <tt class="docutils literal" style="font-family: Consolas, monaco, monospace; color: rgb(0, 0, 0); background-color: rgb(247, 247, 247); white-space: nowrap; border-radius: 2px; font-size: 21.6px; padding: 2px;">libdwarf - Working with DWARF programmatically

Employing command-line tools to access DWARF information, while useful, isn't fully satisfying. As programmers, we'd like to know how to write actual code that can read the format and extract what we need from it.

Naturally, one approach is to grab the DWARF specification and start hacking away. Now, remember how everyone keeps saying that you should never, ever parse HTML manually but rather use a library? Well, with DWARF it's even worse. DWARF is  _much_  more complex than HTML. What I've shown here is just the tip of the iceberg, and to make things even harder, most of this information is encoded in a very compact and compressed way in the actual object file [[6]][23].

So we'll take another road and use a library to work with DWARF. There are two major libraries I'm aware of (plus a few less complete ones):

1.  BFD (libbfd) is used by the [GNU binutils][11], including objdump which played a star role in this article, ld (the GNU linker) and as (the GNU assembler).
2.  libdwarf - which together with its big brother libelf are used for the tools on Solaris and FreeBSD operating systems.

I'm picking libdwarf over BFD because it appears less arcane to me and its license is more liberal (LGPLvs. GPL).

Since libdwarf is itself quite complex it requires a lot of code to operate. I'm not going to show all this code here, but [you can download][24] and run it yourself. To compile this file you'll need to have libelfand libdwarf installed, and pass the -lelf and -ldwarf flags to the linker.

The demonstrated program takes an executable and prints the names of functions in it, along with their entry points. Here's what it produces for the C program we've been playing with in this article:

```
$ dwarf_get_func_addr tracedprog2
DW_TAG_subprogram: 'do_stuff'
low pc  : 0x08048604
high pc : 0x0804863e
DW_TAG_subprogram: 'main'
low pc  : 0x0804863e
high pc : 0x0804865a
```

The documentation of libdwarf (linked in the References section of this article) is quite good, and with some effort you should have no problem pulling any other information demonstrated in this article from the DWARF sections using it.

### Conclusion and next steps

Debugging information is a simple concept in principle. The implementation details may be intricate, but in the end of the day what matters is that we now know how the debugger finds the information it needs about the original source code from which the executable it's tracing was compiled. With this information in hand, the debugger bridges between the world of the user, who thinks in terms of lines of code and data structures, and the world of the executable, which is just a bunch of machine code instructions and data in registers and memory.

This article, with its two predecessors, concludes an introductory series that explains the inner workings of a debugger. Using the information presented here and some programming effort, it should be possible to create a basic but functional debugger for Linux.

As for the next steps, I'm not sure yet. Maybe I'll end the series here, maybe I'll present some advanced topics such as backtraces, and perhaps debugging on Windows. Readers can also suggest ideas for future articles in this series or related material. Feel free to use the comments or send me an email.

### References

*   objdump man page
*   Wikipedia pages for [ELF][12] and [DWARF][13].
*   [Dwarf Debugging Standard home page][14] - from here you can obtain the excellent DWARF tutorial by Michael Eager, as well as the DWARF standard itself. You'll probably want version 2 since it's what gccproduces.
*   [libdwarf home page][15] - the download package includes a comprehensive reference document for the library
*   [BFD documentation][16]


[1] DWARF is an open standard, published here by the DWARF standards committee. The DWARF logo displayed above is taken from that website.

[2] At the end of the article I've collected some useful resources that will help you get more familiar with DWARF, if you're interested. Particularly, start with the DWARF tutorial.

[3] Here and in subsequent examples, I'm placing (...) instead of some longer and un-interesting information for the sake of more convenient formatting.

[4] Because the DW_AT_frame_base attribute of do_stuff contains offset 0x0 into the location list. Note that the same attribute for main contains the offset 0x2c which is the offset for the second set of location expressions.

[5] Where the function prologue is usually executed and the local variables aren't even valid yet.

[6] Some parts of the information (such as location data and line number data) are encoded as instructions for a specialized virtual machine. Yes, really.

* * *

--------------------------------------------------------------------------------

via: http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information

作者：[Eli Bendersky][a]
译者：[译者ID](https://github.com/译者ID)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]:http://eli.thegreenplace.net/
[1]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id1
[2]:http://dwarfstd.org/
[3]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id2
[4]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id3
[5]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id4
[6]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id5
[7]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id6
[8]:http://eli.thegreenplace.net/tag/articles
[9]:http://eli.thegreenplace.net/tag/debuggers
[10]:http://eli.thegreenplace.net/tag/programming
[11]:http://www.gnu.org/software/binutils/
[12]:http://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[13]:http://en.wikipedia.org/wiki/DWARF
[14]:http://dwarfstd.org/
[15]:http://reality.sgiweb.org/davea/dwarf.html
[16]:http://sourceware.org/binutils/docs-2.21/bfd/index.html
[17]:http://en.wikipedia.org/wiki/DWARF
[18]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id7
[19]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id8
[20]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id9
[21]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id10
[22]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id11
[23]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information#id12
[24]:https://github.com/eliben/code-for-blog/blob/master/2011/dwarf_get_func_addr.c
[25]:http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information
[26]:http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/
[27]:http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/