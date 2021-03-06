---
title: "WebAssembly Troubles part 3: The Stack Is Not the Stack"
date: 2019-01-31T11:28:18+01:00
draft: true
---

> ### Preamble
> 
> This is part 3 of a 4-part miniseries on issues with WebAssembly and proposals to fix them. [Part 1 here][part-1], [part 2 here][part-2].
> <br/><br/>
> This article assumes some familiarity with virtual machines, compilers and WebAssembly, but I'll try to link to relevant information where necessary so even if you're not you can follow along.
> <br/><br/>
> Also, this series is going to come off as if I dislike WebAssembly. I love WebAssembly! I wrote a [whole article about how great it is][wasm-on-the-blockchain]! In fact, I love it so much that I want it to be the best that it can be, and this series is me working through my complaints with the design in the hope that some or all of these issues can be addressed soon, while the ink is still somewhat wet on the specification.

[wasm-on-the-blockchain]: {{< ref "/posts/why-wasm.md" >}}
[part-1]: {{< ref "/posts/wasm-is-not-a-stack-machine.md" >}}
[part-2]: {{< ref "/posts/why-do-we-need-the-relooper-algorithm-again.md" >}}

If you read my [first article in this series][part-1], you'll know that WebAssembly has both locals and the stack. I made the argument that locals are not just unnecessary, but are detrimental to the performance of WebAssembly in general, and that everything could be done with the stack alone. However, if you're familiar with LLVM IR or other SSA IRs you might be familiar with an operator known as `alloca`, which circumvents SSA form to create an actual value on the stack. Why does LLVM need that escape hatch while I argue that WebAssembly does not?

Well, here's a thought experiment for you: what WebAssembly does LLVM emit for the following Rust code?

```rust
extern { fn test(_: &mut u32); }

fn main(){
    let mut a = 0;
    unsafe { test(&mut a) };
}
```

In LLVM's IR this is represented using `alloca`, which returns a pointer to the new stack value. In WebAssembly, however, you cannot create pointers to locals, so `alloca` has to be compiled another way. Let's have a look at the emitted WebAssembly when compiled in release mode. I've simplified it by hand to make it clearer what's going on.

```lisp
(module
  (import "env" "test" (func $test (param i32)))
  (func $main
    (local $local_sp i32)
    (global.set $stack_ptr
      (local.tee $local_sp
        (i32.sub
          (global.get $stack_ptr)
          (i32.const 16))))
    (i32.store offset=12
      (local.get $local_sp)
      (i32.const 0))
    (call $test
      (i32.add
        (local.get $local_sp)
        (i32.const 12)))
    (global.set $stack_ptr
      (i32.add
        (local.get $local_sp)
        (i32.const 16))))
  (memory $memory (export "memory") 17)
  (global $stack_ptr (mut i32) (i32.const 1048576)))
```

Gross. LLVM generates a mutable global (similar to `static mut` in Rust) which it then does arithmetic manually on to manipulate the stack. Accessing the stack is suppposed to be extremely fast, and in fact if you compile the Rust code directly to native it looks something like this:

```gas
main:
    push rax
    mov  dword ptr [rsp + 4], 0
    lea  rdi,                 [rsp + 4]
    call qword ptr [rip + test@GOTPCREL]
    pop  rax
    ret
```

The `push` and `pop` are to reserve space on the stack, it's better if you need to reserve <8 bytes to use `push` and `pop` instead of doing arithmetic on the stack pointer yourself. Contrast this assembly output with the assembly generated by firefox for the above Wasm (simplified):

```gas
main:
  sub rsp,                         0x28
  cmp [r14 + 0x28],                rsp
  ; .Lstack-overflow is defined elsewhere
  jae .Lstack-overflow 

  mov edi,                         [r14 + 0x50]
  mov eax,                         edi
  sub eax,                         0x10
  mov [rsp + 0x1c],                eax
  mov [r14 + 0x50],                eax
  mov [r15 + rax + 0xc],           0
  add edi,                         -4
  mov [rsp],                       r14
  mov rax,                         [r14 + 0x30]
  mov r14,                         [r14 + 0x38]
  mov r15,                         [r14 + 0x18]
  call rax
  mov r14,                         [rsp]
  mov r15,                         [r14 + 0x18]
  mov eax,                         [rsp + 0x1c]
  add eax,                         0x10
  mov [r14 + 0x50],                eax
  nop
  add rsp,                         0x28
  ret
```

Some of this is important for WebAssembly's sandboxing, but I hope you can see that the "stack_ptr" global is being loaded and stored to memory every time. This isn't the baseline compiler, this is Firefox's optimising compiler, and as far as I can tell at the time of writing Firefox's output for WebAssembly is the fastest of any browser. "So", you ask, "why not just reserve a register for this global variable?". Well I'm glad you asked, straw man in my head. Although it wouldn't solve every problem, having the stack pointer be stored in a global variable would improve performance significantly and allow for much better code. It still couldn't match native, but it would get a lot closer.

The issue is that the global used for the stack pointer isn't marked. The Wasm->native compiler doesn't know that `stack_ptr` is the stack pointer and so will be hot enough to store in a register. We've got a few ways to solve this, and I'll give my take on the pros and cons of each.

## Separate stack and heap

The first possibility is to have each function declare a set of stack variables and allow it to construct pointers to them. Unfortunately, this is a total non-starter. Currently pointers in Wasm are just a `u32` (Wasm has one type for both signed and unsigned integers so you'll see it referred to as `i32` in the spec), which means that arbitrary arithmetic can be performed on them. If you want the stack pointers to refer to the "physical stack" (the area of memory around `rsp`, like LLVM generates when generating native code) you're out of luck since a malicious Wasm module could just manipulate a stack pointer and overwrite the return address. PNaCl implements this method and prevents the return address from being manipulated by masking the return address when the function returns, which ensures that you can only jump to known-safe addresses. Unfortunately, manipulating the return address like this means that every single function call results in a pipeline stall since the CPU can't start speculatively executing the return value.

That's not the only problem though. Currently, the Wasm heap is untyped, which is very important for efficient compilation of languages like C and Rust that rely on viewing the heap as a blob of bytes. Heap pointers are just indices into a blob of memory. If the heap and stack were separate, there would be no good way to compile this code:

```rust
fn main() {
    let foo = Box::new(0);
    let bar = 1;

    let some_vec = vec![&*foo, &bar];
}
```

Here we're storing one heap pointer and one stack pointer in an array which is itself stored on the heap. Separated stack and heap pointers means that this is impossible - you either need the heap to be typed or you need to allow manipulating the pointer to the stack value, which could cause the wasm to overwrite memory that it shouldn't have access to.

## Explicit stack pointer

The second possibility - which I believe has been suggested before although I can't find the issue where it was mentioned - is to explicitly mark one or more globals as "hot", which can act as a hint to the compiler that a register should be reserved for this global. The stack pointer can then be marked as "hot" and WebAssembly runtimes that don't care about maximising performance (like the spec interpreter or [wasmi][wasmi]) can ignore it. The biggest downside to this scheme is that it uses up one more register that could have been used for intermediate results. If your compiler has decent instruction choice then codegen with this scheme should look relatively similar to the native code generated by LLVM. Unlike more complicated optimisations, I would consider implementing a good instruction choice algorithm to be the main job of a compiler from WebAssembly to native, so it wouldn't require extra, stack pointer-specific work in order to get this to compile to good code - work done improving codegen for this case will improve codegen everywhere.

[wasmi]: https://github.com/paritytech/wasmi

Personally, I believe that the second option is the best.

## Why isn't this implemented already?

Unlike the past two articles, this is pretty clearly not a fundamental design issue that should have be fixed for the MVP. This can easily be added in a backwards-compatible way - existing code can't take advantage of this optimisation but any new code will be able to. Hopefully this article gave you some insight into why the stack isn't quite as optimised as it should be in contemporary WebAssembly runtimes.

Join me next time where I finally inject some positivity into this series, and propose a medium-term solution to these issues and more. I won't even have to fight Google on this one.
