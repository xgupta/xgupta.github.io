---
title: "Supporting Debug Info for TLS Variables on AArch64 in LLVM"
date: 2026-04-01T14:35:52+05:30
draft: false
---


This is my attempt to fix the long standing debug info support for TLS variables for AArch64. And I wanted to document this since it involved everything, Compiler, Assembler, Linker, Debugger, utils like dwarfdump, ABI changes and Compatibility with GCC toolchian.  

### Background -  
TLS stands for Thread Local Storage. These variables have local storage which means each thread gets its unique, independent instance of variable. This was introduced in C++11 as `thread_local` keyword and in C as `_Thread_local` keyword with `<threads.h>`.

They are heavily used in performance-critical software like database systems such as ScyllaDB where developers have reported issues with TLS debugging. 

There have been many reports that TLS variable debugging is not working as expected and this issue has been open from nine years -  
https://gcc.gnu.org/bugzilla/show_bug.cgi?id=83010  
https://gcc.gnu.org/bugzilla/show_bug.cgi?id=119531  
https://gcc.gnu.org/bugzilla/show_bug.cgi?id=97344   
https://sourceware.org/bugzilla/show_bug.cgi?id=28351  
https://github.com/llvm/llvm-project/issues/71666  
https://github.com/llvm/llvm-project/issues/83466  

And there is also an attempt to fix it with only LLDB https://github.com/llvm/llvm-project/pull/110822.  


#### TLS layout -
```
static thread_local unsigned long v = 1; 

int main() { return v; }
```

Thread 1 TLS Block
```
TP (thread pointer)
  ↓
+----------------------+
| v  = 1               |  <-- offset A
+----------------------+
```
Thread 2 TLS Block
```
TP (thread pointer)
  ↓
+----------------------+
| v  = 1               |  <-- offset A
+----------------------+
```
They have same layout but different memory region.
Address Formula for both threads:
`&v = TP + offset_v`.
Only TP changes per thread.

### Last Status of LLVM and GCC -  
Consider the example
```
struct kk {
  kk() = default;
  int x = 1, y = 2, z = 3;
  ~kk() {}
};

namespace a {

  static thread_local kk v1;
  thread_local kk v2;

  void f(int x) {
    v1.x = v2.y = x;
  }

  int g() {
    return v1.x + v2.y;
  }

}

int main() {
  return 0;
}
```

Compiling and linking it with -O0 -g, gcc and clang, Both a::v1 and a::v2 can be examined on x86_64 with these commands:  
```
(gdb) start
(gdb) p a::v1
(gdb) p a::v2
```
However, on aarch64  
```
(gdb) p a::v1
$1 = <optimized out>
(gdb) p a::v2
$2 = {x = 1, y = 2, z = 3}
```

```
(lldb) p a::v1 
error: expression failed to parse:
error: <user expression 0>:1:4: no member named 'v1' in namespace 'a'
a::v1 
~~~^
```

This happened because both GCC and LLVM do not emit the offset with DW_AT_location for TLS variable making it difficult for debugger to guess it. For non-static TLS varible GCC can uses libthread_db to heuristically determine variable location but not static varible it need proper location, without that it says variable have been optimize out. It has access to symbol table but that alone was not sufficient to cover all cases even basic ones.

### Current Status of LLVM 
```
$ bin/clang++ -O0 -g -mllvm --aarch64-emit-debug-tls-location -fuse-ld=lld main.cpp -o main
```
```
$ gdb main
(gdb) p a::v1
$1 = {x = 1, y = 2, z = 3}
(gdb) p a::v2
$2 = {x = 1, y = 2, z = 3}
```
```
$ bin/lldb main 
(lldb) p a::v1
(kk)  (x = 1, y = 2, z = 3)
(lldb) p a::v2
(kk)  (x = 1, y = 2, z = 3)
```

### Implementation Approach 

To address this limitation, we need to encode TLS variable locations in a way that debuggers can evaluate at runtime.

Unlike regular global variables, TLS variables do not have a fixed address. Instead, their address must be computed dynamically as:
```
address = Thread Pointer (TP) + offset
```
This means debug information must describe how to compute the address, not just provide a static location.

So the plan was to emit DW_AT_location which encode DW_OP_form_tls_address and provide a `constant offset` so it looks like this -  
```
DW_AT_location  (DW_OP_const8u 0x0, DW_OP_GNU_push_tls_address)
```
```
$ cat main.cpp                                          
static thread_local unsigned long v = 1; 

int main() { return v; }
```

```
0x00000032:   DW_TAG_variable
                DW_AT_name	("v")
                DW_AT_type	(0x00000046 "unsigned long")
                DW_AT_decl_file	("/llvm-project/build/main.cpp")
                DW_AT_decl_line	(1)
                DW_AT_location	(DW_OP_const8u 0x0, DW_OP_GNU_push_tls_address)
                DW_AT_linkage_name	("_ZL1v")
```

LLVM currently uses DW_OP_GNU_push_tls_address, a GNU extension equivalent to the standard DW_OP_form_tls_address.

The offset encoded in DWARF is represented using the R_AARCH64_TLS_DTPREL64 relocation, this relocation encodes the variable’s offset relative to the thread pointer (DTPREL).

At link time, this relocation is resolved to the correct offset within the TLS block, and the debugger later uses it to reconstruct the variable’s address at runtime.

#### Pull request submitted -   
##### Prerequisites Pull requests -
1 - There is one pull request submitted by Igor Kudrin to support it LLDB by doing the calculation right in DynamicLoaderPOSIXDYLD::GetThreadLocalData.
https://github.com/llvm/llvm-project/commit/a4d786630c4757ce91aef65fc2744fbde650632d, I think that fixes the earlier `no member named 'v1' in namespace 'a'` error that we
seen earlier.  

2 - https://github.com/ARM-software/abi-aa/commit/d455ef1e884fe7a3c4f1e05ed40cdeed9103dea9 for Aarch64 ABI changes by Peter Smith.
It permits the R_AARCH64_TLS_DTPREL dynamic relocation to be used statically so that debug information can use it to describe the location of TLS variables. This was the 
prerequisite. With that we can have R_AARCH64_TLS_DTPREL64 in debug info section like this -

$ bin/llvm-readelf -rW main 
```
Relocation section '.rela.debug_info' at offset 0x548 contains 5 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000008  0000000600000102 R_AARCH64_ABS32        0000000000000000 .debug_abbrev + 0
0000000000000011  0000000900000102 R_AARCH64_ABS32        0000000000000000 .debug_str_offsets + 8
0000000000000015  0000001100000102 R_AARCH64_ABS32        0000000000000000 .debug_line + 0
000000000000001f  0000000d00000102 R_AARCH64_ABS32        0000000000000000 .debug_addr + 8
000000000000003c  0000000400000405 R_AARCH64_TLS_DTPREL64 0000000000000000 _ZL1v + 0
```

This shows that TLS offsets are now preserved in debug sections via relocations, allowing tools like debuggers and dwarfdump to reconstruct the correct runtime address.  

##### Main Pulls requests -
1 - https://github.com/llvm/llvm-project/commit/60c102036acf1508b66b1c3e29ffba10d21a6645
This commit is to support recognising the %dtprel() relocation in assmbely with LLVM assembler which will be emitted by llc/clang in next patch. The %dtprel(v) is the new syntax recently added in https://github.com/llvm/llvm-project/commit/bed89970c3df5e755820708580e405f65ddaa1ba. With this clang can emit 
```
.section .tdata,"awT",@progbits
.skip 8
.globl var
var:
  .word 0

.section        .debug_info,"",@progbits
  .xword  %dtprel(var)
```

And Assembler will assemble it to right relocations
```
 Relocations [
   Section (5) .rela.debug_info {
     0x0 R_AARCH64_TLS_DTPREL64 var 0x0
   }
```

2 - https://github.com/llvm/llvm-project/commit/14ce208a45eb0673f8b28409eec0628ad809923b is added the LLD support, use of it/linker to resolve the R_AARCH64_TLS_DTPREL64 
relocation in final binary. With that linker and assembler support we are ready to emit the debug location with clang.

3 - https://github.com/llvm/llvm-project/commit/8e20a6dc866c54683f801d1ca041c6b3ba302485 This commit adds the support of the debug info by overriding target hook
getDebugThreadLocalSymbol for return `AArch64::S_DTPREL` for Aarch64. With that .debug_info section will contain a `DW_AT_location  (DW_OP_const8u 0x0, DW_OP_GNU_push_tls_address)` entry for TLS variables.

4 - https://github.com/llvm/llvm-project/commit/fa136df3e74c1fd0b838352484aa38c471d21cbd which fixes the following warning when dumping the debug info with `llvm-dwarfdump`.
```
warning: failed to compute relocation: R_AARCH64_TLS_DTPREL64, Invalid data was encountered while parsing the file
```
To fix this warning we have mark the relocation as supported in llvm/lib/Object/RelocationResolver.cpp however also the final absolute address of a TLS variable is determined at runtime, resolving to the symbol's section-relative offset in the object file only mitigate the warning for end user.  


Since GNU binutils do not support this relocation we have also added `aarch64-emit-debug-tls-location` flag to llc will be default to false so that it is explicitly requires
to pass for emitting this debug info until GNU binutils also support it. This is to prevent breaking builds for users who are still using older GNU binutils that don't 
recognize the relocation yet since clang by default uses GNU ld linker unless overridden by -fuse-ld=lld flag.

5 - I have also sent the patch for GNU binutils https://sourceware.org/pipermail/binutils/2026-March/148550.html. This is my first patch to any GNU project and it helps me
to understand BFD library, gnu as, gnu ld and gold. It was new experience since I mostly read LLVM source code. It is still under review and hopefully will get merge in next week.

These changes enable correct debugging of TLS variables on AArch64 by providing end-to-end support across the compiler, assembler, linker, and debug tools. It closes a 
long-standing gap in the toolchain and aligns AArch64 behavior with other architectures like x86_64.

With that I would like to thanks many people who reviewed the changes, Peter Smith, Jessica Clarke, Fangrui Song, Igor Kudrin, Avi Kivity and Alice Carlotti from GNU side.
