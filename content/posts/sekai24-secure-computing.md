+++
title = 'Sekai24 - Secure Computing'
summary = 'Windows Reversing Challenge @ Sekai CTF 2024'
date = 2024-09-25T17:36:34-04:00
tags = ['CTF', 'writeups', 'reverse', 'windows']
draft = true
+++

<!-- draft = true -->

> _REVERSE, 408 (6 solves)_

This was a **Windows** reverse challenge. I didn't solve the challenge during the ctf and faced some issues during the reversing because of Windows version problem that made it impossible to solve.

## Problem Description

We were given a <cite>website[^chal]</cite> (shown in following image), containing several versions of the challenge for different window build versions, and the following description:

> I heard that Windows allows any process to interact with any other process. Good thing all the important parts of my program execute directly in the kernel!
>
> Author: <cite>molenzwiebel[^author]</cite>  
> ![](/images/sekai24-secure-computing/downloads-page.png)

## Reversing

### What does the program do?

Since the author provided multiple versions of the challenge, I simply downloaded the version that matched my Windows Virtual Machine, _Windows 10 21H2_, and tried running.

```
Welcome to my secure crackme, now powered by secure ring-0 encryption services (patent pending).
This executable requires kernel versions 19041, 19042, 19043, or 19044 (Windows 10 2004, 20H2, 21H1, or 21H2) to work correctly. Check your OS version with `winver` and download a different version from the SekaiCTF website if necessary.

Enter the secret code:
asdf

Verifying...
That's incorrect! Sorry :(
```

From the output, we can tell that it was a flag checker and our goal is to figure what input was needed to print out the congratulation text.

> Congratulations! Submit as SEKAI{<your input here>} to receive your reward!

### Disassemble

Disassemble the binary, we can find the following code.

```nasm {linenos=table,hl_lines=[6,12,17,23],linenostart=1}
; dist-19041.exe

401000:	mov    rbx,QWORD PTR gs:0x60
401009:	mov    rbx,QWORD PTR [rbx+0x20]
40100d:	mov    rbx,QWORD PTR [rbx+0x28]
401011:	movabs rax,0x1f5168f5b2838008
40101b:	movabs rdx,0x0
401025:	mov    r10,rbx
401028:	movabs r9,0x0
401032:	movabs r8,0x0
40103c:	movabs rsp,0xa14ee0
401046:	syscall
401048:	mov    rsi,rax
40104b:	mov    r12,QWORD PTR gs:0x60
401054:	mov    r12,QWORD PTR [r12+0x20]
401059:	mov    r12,QWORD PTR [r12+0x20]
40105e:	movabs rax,0x5fdac94ae10f4006
401068:	mov    r10,r12
40106b:	movabs r8,0x0
401075:	movabs rdx,0x0
40107f:	movabs r9,0x0
401089:	movabs rsp,0xa14f30
401093:	syscall
401095:	mov    r14,rax

...
```

From the highlighted lines, we can conclude that it is messing around(?) the **Windows System Call**s. Since system call numbers change based on the Windows build version, the first task was to figure out which system call is being called.

### Try and Fail during the CTF

#### Find the corresponding Syscall

I found <cite>Windows System Call Table[^table]</cite> that list out the corresponding all the syscalls and its SSN for each version. Therefore, the first thing was to figure out what windows functions actually being used.

Though `rax` is 64-bit, for all the syscall number, only the **lower 9 bits** matter. That is, even though there seem to have a lot of different values, it should only be calling a certain subset of functions. By simple pattern matching(todo: add link to code) on the disassembled code, we can found that it only called **12** different syscalls:

| Syscall Number |          Syscall Symbol           |
| :------------: | :-------------------------------: |
|     0x006      |           `NtReadFile`            |
|     0x008      |           `NtWriteFile`           |
|     0x02c      |       `NtTerminateProcess`        |
|     0x03a      |      `NtWriteVirtualMemory`       |
|     0x03f      |       `NtReadVirtualMemory`       |
|     0x043      |           `NtContinue`            |
|     0x0a1      |          `NtContinueEx`           |
|     0x0ad      |      `NtCreateIoCompletion`       |
|     0x0c0      |        `NtCreateSemaphore`        |
|     0x0cd      |      `NtCreateWorkerFactory`      |
|     0x150      | `NtQueryInformationWorkerFactory` |
|     0x1a0      |  `NtSetInformationVirtualMemory`  |

Some of the syscalls could be easily figure out but for some of them, I basically had no idea what they were doing.

- `NtReadFile` / `NtWriteFile`: STDIN / STDOUT (both have `FileHandle = 0`)
- `NtTerminateProcess`: Exit program with different status
- <cite>`NtWriteVirtualMemory` / `NtReadVirtualMemory`[^undoc]</cite>

  ```c
  NTSYSAPI NTSTATUS NTAPI NtWriteVirtualMemory(
  	IN HANDLE ProcessHandle,
  	IN PVOID BaseAddress, 						// dst
  	IN PVOID Buffer, 						// src
  	IN ULONG NumberOfBytesToWrite, 					// length
  	OUT PULONG NumberOfBytesWritten OPTIONAL
  );

  NTSYSAPI NTSTATUS NTAPI NtReadVirtualMemory(
  	IN HANDLE ProcessHandle,
  	IN PVOID BaseAddress, 						// src
  	OUT PVOID Buffer, 						// dst
  	IN ULONG NumberOfBytesToRead, 					// length
  	OUT PULONG NumberOfBytesReaded OPTIONAL
  );
  ```

  > Both `memcpy` but `dst`/`src` come from different arguments.

- `NtSetInformationVirtualMemory`

  > Modify(?) information for virual memory (not entirely sure, this never succeed)

- <cite>`NtContinue`[^undoc]</cite> / `NtContinueEx`

  ```c
  NTSYSAPI NTSTATUS NTAPI NtContinue(
  	IN PCONTEXT ThreadContext,
  	IN BOOLEAN RaiseAlert
  );
  ```

  `NtContinue` will restore the thread context from the given `ThreadContext`.

- `NtCreateIoCompletion` / `NtCreateWorkerFactory`
- `NtQueryInformationWorkerFactory`
  > I had no idea what they were and could not figure what they were doing even running dynamically. Figure they must have something to do with Worker Factory, since some code created them. However, this syscall should just be getting the value and had no way of modifying it?
- `NtCreateSemaphore`
  > <s>_(didn't really try to figure this, had already moved to other challenges...)_</s>

With `NtContinue` and `NtContinueEx`, I figured I would be able to recover control flow (todo: add link to cfg.svg) and want to see if could cheese the challenge. However, no matter what input it would always reach `label_85025f` and based on the condition, it would then decide whether the input was correct and I had no idea what it checked...

### After Contest

## Decompile

## To be Continue... (probably not?)

> Other thoughts of how I would approach this challenge but I would probably not implement it...

### Alternative Solution #1

After lifting to python code, I found that the computation was really simple and should(?) be **_Z3_**-able or at least do something smarter. However, since bruteforce already worked, I was too lazy to solve it this way.

Computation consists of a lot of following chunks,

```py
...
# treat rsi as input
r15 = 'id_0'; wf[r15] = 0
rdi = 'id_1'; wf[rdi] = 0
if rsi:
	wf[rdi] = (wf[rdi] + rsi) & 0xffff
if rsi & 0x8000:
	wf[rdi] = (wf[rdi] + 0x8000) & 0xffff
	rsi = wf[rdi]
	wf[r15] = (wf[r15] + 0x0100) & 0xffff
if rsi & 0x4000:
	wf[rdi] = (wf[rdi] + 0xc000) & 0xffff
	rsi = wf[rdi]
	wf[r15] = (wf[r15] + 0x0080) & 0xffff
if rsi & 0x2000:
	wf[rdi] = (wf[rdi] + 0xe000) & 0xffff
	rsi = wf[rdi]
	wf[r15] = (wf[r15] + 0x0040) & 0xffff
...
if rsi & 0x0002:
	wf[r15] = (wf[r15] + 0x0400) & 0xffff
	wf[rdi] = (wf[rdi] + 0xfffe) & 0xffff
	rsi = wf[rdi]
if rsi:
	wf[rdi] = (wf[rdi] + 0xffff) & 0xffff
	rsi = wf[rdi]
	wf[r15] = (wf[r15] + 0x0200) & 0xffff
...
```

For each of the chunks, it was basically applied a simple calculation on the input (some may be calculted based on two inputs). Therefore, it can be transformed into the following form.

```py
...
# treat rsi as input
# and wf[r15] is the result
result = 0
result = If(rsi & 0x8000, result | 0x0100, result)
result = If(rsi & 0x4000, result | 0xc000, result)
result = If(rsi & 0x2000, result | 0x0040, result)
...
result = If(rsi & 0x0002, result | 0x0400, result)
result = If(rsi & 0x0001, result | 0x0200, result)
...
```

As shown above, there was only `and`/`or` opertions on 16-bit numbers.

### Alternative Solution #2

Instead of going through all the trouble of generating runnable python code, an easier and faster way should be bruteforcing the solution by having <u>window debuggers scripts</u> or <u>patching the binary</u>. However, I was solving it after the contest ended (totally _no time constraints_ and absolutely have _spend way toooooo much time_) and was trying to challenge myself to actually decompile the challenge.

# Related Links

- Source Code for Challenge: [sekaictf-syscall-compiler](https://github.com/molenzwiebel/sekaictf-syscall-compiler)
- Author's Solution: [gist](https://gist.github.com/molenzwiebel/0e8248f89079cf192e52673f6a8dfa5a)
- [Syscall Table](https://j00ru.vexillium.org/syscalls/nt/64/)
- Windows Structures: [VERGILIUS](https://www.vergiliusproject.com/kernels/x64)
- [NTAPI Undocumented Functions](http://undocumented.ntinternals.net/index.html)
- [ReactOS](https://doxygen.reactos.org/search.html?query=ntcontinue)

[^chal]: Challenge [Website](https://2024.ctf.sekai.team/files/7f0170c9b7ef4562e7c4bac3cf76a18b/instructions.html): <i>Secure Computing - Downloads</i>
[^author]: [GitHub](https://github.com/molenzwiebel) for Challenge Author
[^table]: [Website](https://j00ru.vexillium.org/syscalls/nt/64/) that contains a collection of system call tables for multiple releases.
[^undoc]: [NTAPI Undocumented Functions](http://undocumented.ntinternals.net/index.html)
[^vergilius]: Windows Kernel Undocumented Structures [VERGILIUS](https://www.vergiliusproject.com/kernels/x64)
