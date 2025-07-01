---
layout: post
title:  "HackTheBox Heartbreaker-Continuum Malware Analysis Writeup"
date:   2024-11-29 +0800
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

**Answer:**

# Question 4

**Question:** What is the API that used malware-allocated memory to write shellcode?

**Answer:**

# Question 5

**Question:** What is the protection of allocated memory?

**Answer:**

# Question 6

**Question:** What assembly instruction is used to transfer execution to the shellcode?

**Answer:**

# Question 7

**Question:** What is the number of functions the malware resolves from `kernel32`?

**Answer:**

# Question 8

**Question:** The malware obfuscates two strings after calling `RegisterClassExA`. What is the first string?

**Answer:**

# Question 9

**Question:** What is the value of `dwCreationFlags` of `CreateProcessA`?

**Answer:**

# Question 10

**Question:** The Malware uses a process injection technique. What is its name?

**Answer:**

# Question 11

**Question:** What is the API used to write the payload into the target process?

**Answer:**