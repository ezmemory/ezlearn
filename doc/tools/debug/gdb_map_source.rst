How to point GDB to your sources
April 30, 2017
So, you have a binary that you or someone developed and, surprise, it has some bug. Or you just curious how it’s working. Great tool to help with these cases is a debugger.

It’s really seldom when you want to debug on assembly level, usually, you want to see the sources. But often times you debug the program on the host other than the build host and see this really frustrating message:

$ gdb -q python3.7
Reading symbols from python3.7...done.
(gdb) l
6       ./Programs/python.c: No such file or directory.
Ouch. Everybody was here. I’ve seen this so often while it’s so vital for sensible debugging so I think it’s very important to get into details and understand how GDB shows source code in debugging session.

Debug info
It all starts with debug info - special sections in the binary file produced by the compiler and used by the debugger and other handy tools.

In GCC there is well-known -g flag for that. Most projects with some kind of build system either build with debug info by default or have some flag for it.

In the case of CPython, -g is added by default but nevertheless, we’re better off adding --with-pydebug to enable all kinds of debug options available in CPython:

$ ./configure --with-pydebug
$ make -j
While you’re watching the compilation log, notice the -g option in gcc invocations.

This -g option will generate debug sections - binary sections to insert into program’s binary. These sections are usually in DWARF format. For ELF binaries these debug sections have names like .debug_*, e.g. .debug_info or .debug_loc. These debug sections are what makes the magic of debugging possible - basically, it’s a mapping of assembly level instructions to the source code.

To find whether your program has debug symbols you can list the sections of the binary with objdump:

$ objdump -h ./python

python:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .interp       0000001c  0000000000400238  0000000000400238  00000238  2**0
  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.ABI-tag 00000020  0000000000400254  0000000000400254  00000254  2**2
  CONTENTS, ALLOC, LOAD, READONLY, DATA
  ...
  25 .bss          00031f70  00000000008d9e00  00000000008d9e00  002d9dfe  2**5
  ALLOC
  26 .comment      00000058  0000000000000000  0000000000000000  002d9dfe  2**0
  CONTENTS, READONLY
  27 .debug_aranges 000017f0  0000000000000000  0000000000000000  002d9e56  2**0
  CONTENTS, READONLY, DEBUGGING
  28 .debug_info   00377bac  0000000000000000  0000000000000000  002db646  2**0
  CONTENTS, READONLY, DEBUGGING
  29 .debug_abbrev 0001fcd7  0000000000000000  0000000000000000  006531f2  2**0
  CONTENTS, READONLY, DEBUGGING
  30 .debug_line   0008b441  0000000000000000  0000000000000000  00672ec9  2**0
  CONTENTS, READONLY, DEBUGGING
  31 .debug_str    00031f18  0000000000000000  0000000000000000  006fe30a  2**0
  CONTENTS, READONLY, DEBUGGING
  32 .debug_loc    0034190c  0000000000000000  0000000000000000  00730222  2**0
  CONTENTS, READONLY, DEBUGGING
  33 .debug_ranges 00062e10  0000000000000000  0000000000000000  00a71b2e  2**0
  CONTENTS, READONLY, DEBUGGING
  or readelf:

  $ readelf -S ./python
There are 38 section headers, starting at offset 0xb41840:

  Section Headers:
  [Nr] Name              Type             Address           Offset
  Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
  0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
  000000000000001c  0000000000000000   A       0     0     1

  ...

  [26] .bss              NOBITS           00000000008d9e00  002d9dfe
  0000000000031f70  0000000000000000  WA       0     0     32
  [27] .comment          PROGBITS         0000000000000000  002d9dfe
  0000000000000058  0000000000000001  MS       0     0     1
  [28] .debug_aranges    PROGBITS         0000000000000000  002d9e56
  00000000000017f0  0000000000000000           0     0     1
  [29] .debug_info       PROGBITS         0000000000000000  002db646
  0000000000377bac  0000000000000000           0     0     1
  [30] .debug_abbrev     PROGBITS         0000000000000000  006531f2
  000000000001fcd7  0000000000000000           0     0     1
  [31] .debug_line       PROGBITS         0000000000000000  00672ec9
  000000000008b441  0000000000000000           0     0     1
  [32] .debug_str        PROGBITS         0000000000000000  006fe30a
  0000000000031f18  0000000000000001  MS       0     0     1
  [33] .debug_loc        PROGBITS         0000000000000000  00730222
  000000000034190c  0000000000000000           0     0     1
  [34] .debug_ranges     PROGBITS         0000000000000000  00a71b2e
  0000000000062e10  0000000000000000           0     0     1
  [35] .shstrtab         STRTAB           0000000000000000  00b416d5
  0000000000000165  0000000000000000           0     0     1
  [36] .symtab           SYMTAB           0000000000000000  00ad4940
  000000000003f978  0000000000000018          37   8762     8
  [37] .strtab           STRTAB           0000000000000000  00b142b8
  000000000002d41d  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
  as we see in our fresh compiled Python - it has .debug_* section, hence it has debug info.

Debug info is a collection of DIEs - Debug Info Entries. Each DIE has a tag specifying what kind of DIE it is and attributes that describes this DIE - things like variable name and line number.

How GDB finds source code
To find the sources GDB parses .debug_info section to find all DIEs with tag DW_TAG_compile_unit. The DIE with this tag has 2 main attributes DW_AT_comp_dir (compilation directory) and DW_AT_name - path to the source file. Combined they provide the full path to the source file for the particular compilation unit (object file).

To parse debug info you can again use objdump:

  $ objdump -g ./python | vim -
  and there you can see the parsed debug info:

Contents of the .debug_info section:

Compilation Unit @ offset 0x0:
  Length:        0x222d (32-bit)
  Version:       4
  Abbrev Offset: 0x0
  Pointer Size:  8
  <0><b>: Abbrev Number: 1 (DW_TAG_compile_unit)
  <c>   DW_AT_producer    : (indirect string, offset: 0xb6b): GNU C99 6.3.1 20161221 (Red Hat 6.3.1-1) -mtune=generic -march=x86-64 -g -Og -std=c99
  <10>   DW_AT_language    : 12  (ANSI C99)
  <11>   DW_AT_name        : (indirect string, offset: 0x10ec): ./Programs/python.c
  <15>   DW_AT_comp_dir    : (indirect string, offset: 0x7a): /home/avd/dev/cpython
  <19>   DW_AT_low_pc      : 0x41d2f6
  <21>   DW_AT_high_pc     : 0x1b3
  <29>   DW_AT_stmt_list   : 0x0
It reads like this - for address range from DW_AT_low_pc = 0x41d2f6 to DW_AT_low_pc + DW_AT_high_pc = 0x41d2f6 + 0x1b3 = 0x41d4a9 source code file is the ./Programs/python.c located in /home/avd/dev/cpython. Pretty straightforward.

So this is what happens when GDB tries to show you the source code:

 * parses the .debug_info to find DW_AT_comp_dir with DW_AT_name attributes for the current object file (range of addresses)
 * opens the file at DW_AT_comp_dir/DW_AT_name
 * shows the content of the file to you

How to tell GDB where are the sources
  So to fix our problem with ./Programs/python.c: No such file or directory. we have to obtain our sources on the target host (copy or git clone) and do one of the following:

  1. Reconstruct the sources path
  You can reconstruct the sources path on the target host, so GDB will find the source file where it expects. Stupid but it will work.

  In my case, I can just do git clone https://github.com/python/cpython.git /home/avd/dev/cpython and checkout to the needed commit-ish.

  2. Change GDB source path
  You can direct GDB to the new source path right in the debug session with directory <dir> command:

  (gdb) list
  6  ./Programs/python.c: No such file or directory.
  (gdb) directory /usr/src/python
  Source directories searched: /usr/src/python:$cdir:$cwd
  (gdb) list
  6  #ifdef __FreeBSD__
  7  #include <fenv.h>
  8  #endif
  9  
  10 #ifdef MS_WINDOWS
  11 int
  12 wmain(int argc, wchar_t **argv)
  13 {
  14     return Py_Main(argc, argv);
  15 }
  3. Set GDB substitution rule
  Sometimes adding another source path is not enough if you have complex hierarchy. In this case you can add substitution rule for source path with set substitute-path GDB command.

  (gdb) list
  6  ./Programs/python.c: No such file or directory.
  (gdb) set substitute-path /home/avd/dev/cpython /usr/src/python
  (gdb) list
  6  #ifdef __FreeBSD__
  7  #include <fenv.h>
  8  #endif
  9  
  10 #ifdef MS_WINDOWS
  11 int
  12 wmain(int argc, wchar_t **argv)
  13 {
  14     return Py_Main(argc, argv);
  15 }
  4. Move binary to sources
  You can trick GDB source path by moving binary to the directory with sources.

  mv python /home/user/sources/cpython
  This will work because GDB will try to look for sources in the current directory ($cwd) as the last resort.

  5. Compile with -fdebug-prefix-map
  You can substitute the source path on the build stage with -fdebug-prefix-map=old_path=new_path option. Here is how to do it within CPython project:

  $ make distclean    # start clean
  $ ./configure CFLAGS="-fdebug-prefix-map=$(pwd)=/usr/src/python" --with-pydebug
  $ make -j
  And now we have new sources dir:

  $ objdump -g ./python
  ...
  <0><b>: Abbrev Number: 1 (DW_TAG_compile_unit)
  <c>   DW_AT_producer    : (indirect string, offset: 0xb65): GNU C99 6.3.1 20161221 (Red Hat 6.3.1-1) -mtune=generic -march=x86-64 -g -Og -std=c99
  <10>   DW_AT_language    : 12       (ANSI C99)
  <11>   DW_AT_name        : (indirect string, offset: 0x10ff): ./Programs/python.c
  <15>   DW_AT_comp_dir    : (indirect string, offset: 0x558): /usr/src/python
  <19>   DW_AT_low_pc      : 0x41d336
  <21>   DW_AT_high_pc     : 0x1b3
  <29>   DW_AT_stmt_list   : 0x0
  ...
This is the most robust way to do it because you can set it to something like /usr/src/<project>, install sources there from a package and debug like a boss.

[Z] There are situations when we only have the debug libs (*.so), but we don't know full source path for the libs, which is needed to set-substitute. We can find the source path with objdump:
 $ objdump -g /path/to/a/libXYZ.so.1.1.1  | grep DW_AT_comp_dir
<27c5e0>   DW_AT_comp_dir    : (indirect string, offset: 0x85): /scratch/_temp/conan/.conan/data/path/to/source/XYZ

or: using vim to open the output and search for DW_TAG_compile_unit:
$objdump -g /path/to/a/libXYZ.so.1.1.1  | vim -

Then in vim, looking for "DW_TAG_compile_unit", will see sth like:
119279 Contents of the .debug_info section:
119280 
119281   Compilation Unit @ offset 0x0:
119282    Length:        0x28db (32-bit)
119283    Version:       4
119284    Abbrev Offset: 0x0
119285    Pointer Size:  8
119286  <0><b>: Abbrev Number: 1 (DW_TAG_compile_unit)
119287     <c>   DW_AT_producer    : (indirect string, offset: 0x7fc): GNU C++14 8.5.0 20210514 (Red Hat 8.5.0-10) -m64 -mtune=generic -march=x86-64 -g -std=c++14 -fPIC
119288     <10>   DW_AT_language    : 4        (C++)
119289     <11>   DW_AT_name        : (indirect string, offset: 0xa91): /scratch/_temp/conan/.conan/data/path/to/XYZ/MyString.cpp
119290     <15>   DW_AT_comp_dir    : (indirect string, offset: 0x85): /scratch/_temp/conan/.conan/data/path/to/XYZ
119291     <19>   DW_AT_ranges      : 0x30
119292     <1d>   DW_AT_low_pc      : 0x0
119293     <25>   DW_AT_stmt_list   : 0x0
119294  <1><29>: Abbrev Number: 2 (DW_TAG_namespace)


Conclusion
GDB uses debug info stored in DWARF format to find source level info. DWARF is pretty straightforward format - basically, it’s a tree of DIEs (Debug Info Entries) that describes object files of your programs along with variables and functions.

There are multiple ways to help GDB find sources, where the easiest ones are directory and set substitute-path commands, though -fdebug-prefix-map is really useful.

Now, when you have source level info go and explore something!

Resources
Introduction to the DWARF Debugging Format
GDB doc on source path
