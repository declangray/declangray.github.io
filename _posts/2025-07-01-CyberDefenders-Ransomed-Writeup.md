---
layout: post
title:  "CyberDefenders Ransomed Malware Analysis Writeup"
date:   2025-07-01 +0800
categories: write-up
---
<link rel="stylesheet" href="/assets/main.css">

# Table of Contents
1. [Introduction](#introduction)
2. [Question 1](#question-1)
3. [Question 2](#question-2)
4. [Question 3](#question-3)
5. [Question 4](#question-4)
6. [Question 5](#question-5)
7. [Question 6](#question-6)
8. [Question 7](#question-7)
9. [Question 8](#question-8)
10. [Question 9](#question-9)
11. [Question 10](#question-10)
12. [Question 11](#question-11)
13. [Conclusion](#conclusion)

# Introduction
This is a writeup for the [Ransomed](https://cyberdefenders.org/blueteam-ctf-challenges/ransomed/) CyberDefenders lab. This is a **hard** difficulty malware analysis lab, coming into this you're going to want some experience with debugging, as well as some understanding of Assembly, and the Win32 API. 

As this is sample targets Windows machines, I will be using [FLARE](https://github.com/mandiant/flare-vm), a Windows-based malware reverse engineering environment from Google's Mandiant, for the analysis of this sample. Using a Windows-based reverse engineering environment allows us to perform in-depth dynamic analysis, including debugging which will be essential to completing this lab.

To begin, download the password-protected zip file `87-Ransomed.zip` into your analysis environment. Unzip it using the provided password and you can now begin analysis.

# Question 1

**Question:** What is the value of entropy?

*Entropy* is a measure of the randomness of the data within a file. Higher values indicate that the sample may be packed or encrypted - techniques often used to avoid detection by Anti-Virus solutions. In order to find the entropy of this sample, we can use a tool called *Detect-It-Easy* - which comes pre-installed with FLARE. 

Opening the sample within Detect-It-Easy we can easily view the entropy by selecting the `Advanced` checkbox and clicking the `Entropy` button.

![Detect-It-Easy entropy page.](/assets/images/Ransomed/entropy.png)

From here we can see the entropy (and thus, the answer) is `7.67720`, Detect-It-Easy also helpfuly indicates that the sample has a 95% chance of being packed. 

**Answer:** 7.677

# Question 2

**Question:** What is the number of sections?

Staying within Detect-It-Easy, we can navigate to the `Sections` tab in order to view the sections of the executable. Sections contain different data used by the executable. In a Windows PE32 file (a standard Windows executable), the `.text` section typically contains the code for the executable, while the `.data` section contains any constant variables.

![Detect-It-Easy sections page.](/assets/images/Ransomed/sections.png)

Looking within Detect-It-Easy we can clearly see 4 sections.

**Answer:** 4

# Question 3

**Question:** What is the name of the technique used to obfuscate string?

Many detection methods rely on the presence of certain strings within an executable, such as API imports, domains, and other strings unique to malicious samples. Due to this, malware authors often implement measures to obfuscate strings. This makes static string analysis challenging, and forces us to use other, more dynamic methods in order to discern strings from the malware. Common string obfuscation methods include Base64 encoding, XOR encoding, and stack strings.

Loading the sample into Ghidra we can view the disassembly of the executable. Looking at the import list within Ghidra, we can see that the sample calls `LoadLibraryW` from `kernel32.dll`.

![Import list highlighting LoadLibraryW](/assets/images/Ransomed/imports.png)

This function is used to import further API functions during runtime. If we go to where this function is first used, at the address `0x0049c870` and look just before it is called we can see two functions: `FUN_0049bee0` and `FUN_0049bfa0`.

Inspecting the first function, `FUN_0049bee0`, we can see some interesting activity.

![Putting string 'kernel32.dll' onto stack](/assets/images/Ransomed/getmodulehandle.png)
After converting the `dword` values to 'Char Sequence' we can see the function putting the string 'kernel32.dll' onto the stack using several `MOV` calls. It then calls `GetModuleHandleA` in order to get a handle to `kernel32.dll`.

![Putting string 'VirtualProtect' onto stack](/assets/images/Ransomed/getprocaddress.png)
Following this, again converting the `dword` values to 'Char Sequence', we can see the function putting the string 'VirtualProtect' onto the stack and then call `GetProcAddress` in order to retrieve the address of the API function `VirtualProtect` within the `kernel32.dll` module.

Here we can see the sample load characters onto the stack in order to obfuscate API calls. The `VirtualProtect` API function allows a process to change the memory protection of memory sections within itself, and can be used by malware to decrypt or unpack additional functionality.

This process of moving characters onto the stack to create strings is known as 'Stack Strings' and gives us our answer.

**Answer:** Stack Strings

# Question 4

**Question:** What is the API that used malware-allocated memory to write shellcode?

We just observed the malware utilising stack strings to load the `VirtualProtect` API to modify the permissions of memory. According to the [documentation](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualprotect) for this function, `VirtualAlloc` and `VirtualAllocEx` can be called in order to allocate the memory space to then call `VirtualProtect` upon. These are the most likely functions that the malware would use in order to allocate memory to write shellcode to.

Opening the executable up within x32dbg, we can then set breakpoints on the `VirtualAlloc` and `VirtualAllocEx` functions to see if these are called.

![Setting breakpoints on VirtualAlloc functions](/assets/images/Ransomed/virtualalloc-breakpoints.png)

We can then run the sample within x32dbg until one of our breakpoints are (hopefuly) hit.

![VirtualAlloc breakpoint hit](/assets/images/Ransomed/virtualalloc-breakpoint-hit.png)

Success! We can see the malware calling `VirtualAlloc`. This is our answer. 

**Answer:** VirtualAlloc

# Question 5

**Question:** What is the protection of allocated memory?

Now that we've hit our breakpoint on `VirtualAlloc` we can follow it to figure out the address that is allocated.

After stepping into the `VirtualAlloc` function in our debugger, we can set a breakpoint where this function returns on `RET 10`

![Breakpoint on return](/assets/images/Ransomed/return-breakpoint.png)

Once this breakpoint is hit, we can see the starting address returned to the `EAX` register.

![Memory address](/assets/images/Ransomed/memory-address.png)

In this case the address is `02730000`. We can now navigate to the memory map tab of x32dbg in order to view the protection on this address.

![Memory protection](/assets/images/Ransomed/memory-protection.png)
Here we can see the protection on this section of memory is set to `ERW`, or execute, read, and write.

**Answer:** ERW

# Question 6

**Question:** What assembly instruction is used to transfer execution to the shellcode?

Now that the memory has been allocated we can start to look for a jump to it. Stepping through the exection from our previous breakpoint on the return call from `VirtualAlloc`, a few instructions ahead we can see a jump.

![Shellcode jump instruction](/assets/images/Ransomed/shellcode-jump.png)

Here we can see the instruction `jmp dword ptr ss:[ebp-4]` being used to transfer exection to the shellcode. 

**Answer:** jmp dword ptr ss:[ebp-4]

# Question 7

**Question:** What is the number of functions the malware resolves from `kernel32`?

To obtain the *actual* malware, we can first set a breakpoint on the previously identified function to transfer execution to the shellcode `jmp dword ptr ss:[ebp-4]`. It may also be an idea to follow the allocated memory in the dump, in order to view when data has been written to this section.

Letting the executable run until it hits the previously set breakpoint, we can now see that the previously allocated memory now contains data.

![The memory space now contains shellcode](/assets/images/Ransomed/shellcode-data.png)

If we then head back to the memory map and navigate to this memory space, we can right-click on the section and select 'Dump Memory to file'. We now have a `.bin` file containing the shellcode.

We can then use the tool scdbg to debug this shellcode.

Simply running the tool with `scdbg /f shellcode.bin` will reveal the functions resolved from various libraries.

![Functions imported by the shellcode](/assets/images/Ransomed/kernel32-functions.png)

Counting from below where `kernel32` is loaded (`LoadLibraryA(kernel32)`) we can count 16 functions.

**Answer:** 16

# Question 8

**Question:** The malware obfuscates two strings after calling `RegisterClassExA`. What is the first string?

Back to x32dbg, we can set a breakpoint for calls to `RegisterClassExA`. If we then let the executable run until it hits this breakpoint we can then slowly step through until we observe some obfuscated strings.

![Obfuscated string identified](/assets/images/Ransomed/obfuscated-strings.png)

We can clearly see the string 'saodkfnosa9uin' which gives us our answer.

**Answer:** saodkfnosa9uin

# Question 9

**Question:** What is the value of `dwCreationFlags` of `CreateProcessA`?

We can set another breakpoint for calls to `CreateProcessA` and then observe stack values in order to identify the passed arguments.

![Stack values when CreateProcessA is called](/assets/images/Ransomed/CreateProcess-stack.png)

We can then cross-reference the stack against the [documentation](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa) for this function. Doing this we can see the following values:

```
Parameter                   Value in stack                    
lpApplicationName           0x008F0000
lpCommandLine               0x00A24358
lpProcessAttributes         0x00000000
lpThreadAttributes          0x00000000
bInheritHandles             0x00000000
dwCreationFlags             0x08000004
lpEnvironment               0x00000000
lpCurrentDirectory          0x00000000
lpStartupInfo               0x007EB7A0
lpProcessInformation        0x007EB7F8
```

Which gives us our answer, `0x08000004`

**Answer:** 0x08000004

# Question 10

**Question:** The Malware uses a process injection technique. What is its name?

Reading further into the `CreateProcessA` documentation, we can determine what the value of `dwCreationFlags` actually means in the context of creating a new process. 

The Microsoft documentation for [process creation flags](https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags) is useful here. The value `0x08000000` is associated with the flag `CREATE_NO_WINDOW` which just simply creates a process without a GUI window, not much use for figuring out the injection technique. However, the `0x00000004` value is associated with the flag `CREATE_SUSPENDED` which is interesting.

So now we know that the malware spawns a new process *in a suspended state*, allocates memory inside of it, and then injects shellcode into the allocated space. These are the hallmarks of *process hollowing* where memory is hollowed out in a suspended process for injection. This process is described in the MITRE ATT&CK framework as [T1055.012](https://attack.mitre.org/techniques/T1055/012/).

**Answer:** Process Hollowing

# Question 11

**Question:** What is the API used to write the payload into the target process?

Referring back to the MITRE ATT&CK technique for process hollowing, it mentions the use of `ProcessWriteMemory` to write to the memory of the suspended process. We can set a breakpoint for any calls to `ProcessWriteMemory` in x32dbg to check if this is used in our sample.

![The process hits the breakpoint on ProcessWriteMemory](/assets/images/Ransomed/writeprocessmemory.png)

Ta-da! The sample hit our breakpoint, so it would appear this is the API being used to write the shellcode to the suspended process.

**Answer:** ProcessWriteMemory

# Conclusion
CyberDefenders was recommended to me quite recently and so far I've really been enjoying their malware analysis labs. This lab was a really great experience and I learned quite a lot about debugging. I hope to do some more write-ups for these CyberDefenders labs in the future.