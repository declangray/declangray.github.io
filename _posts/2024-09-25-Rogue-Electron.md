---
layout: post
title: "Rogue Electron: Exploiting Electron Applications"
date: 2024-09-25 +8000
categories: development
---
<link rel="stylesheet" href="/assets/main.css">
<script src="/assets/js/oneko.js"></script>


## Introduction
### LOLBASs
Recently, I was researching [LOLBASs](https://lolbas-project.github.io/#) (Living off the land binaries, scripts and libraries (*yes the acronym technically does not match...*)) which are binaries (*and scripts and libraries*) already installed on Windows systems that can be used for nefarious purposes. A common example is the DLL `comsvcs.dll`, which can be used to dump the LSASS process for extracting passwords.

Interestingly, I noticed that [Microsoft Teams (Classic)](https://lolbas-project.github.io/lolbas/OtherMSBinaries/Teams/) was listed as a LOLBAS (while Teams is not installed by default, the LOLBAS project also includes applications that are *typically* installed). One of the ways to exploit Teams stuck out to me, replacing a file called `app.asar`. The LOLBAS project referenced [this](https://l--k.uk/2022/01/16/microsoft-teams-and-other-electron-apps-as-lolbins/) blog post from security researcher Andrew Kisliakov.

### ASAR Archives
In Andrew's blog post, he notes that the exploit works on *almost* all Electron applications, but focuses on MS Teams as it is a Microsoft-signed executable (good for [binary proxy execution](https://attack.mitre.org/techniques/T1218/)) and therefore considered a LOLBAS. The Electron Framework allows developers to make desktop applications from their web apps, essentially just stuffing them into a Chrome browser and churning out an executable.

The exploit detailed by Andrew involves creating an "ASAR archive", which is an archive used by Electron apps to store their source code, and placing it within the application's directory. When the application is executed, the code within the ASAR archive will be run.

Because Electron apps are just glorified websites, the ASAR archives largely include JavaScript, which can be modified to whatever you want. The PoC provided by Andrew spawns calc.exe (as all great PoCs do).

`require('child_process').spawn('calc.exe');`

A JSON file is also required, telling the Electron application which script is the "main" script. Assuming you called your calc spawning JavaScript file `main.js`, your JSON file (`package.json`) would contain:

`{ "main" : "main.js" }`

Place these two files, `main.js` and `package.json` into a folder called `app`, then using the npx tool (which comes with npm), you can create an "ASAR archive" with the command:

`npx asar p app app.asar`

This will create an ASAR archive called `app.asar` which contains your two files. Replacing the actual `app.asar` file of *most* Electron apps will result in them opening calc.exe instead of doing what they'd normally do (in the case of MS Teams this is an improvement).

### Testing
So of course, after discovering this I decided to test this on as many Electron applications as I could get my hands on.

There are *a lot* of Electron applications, in fact if you go to Electron's website they have a [very long list](https://www.electronjs.org/apps) of featured Electron applications. I'm not going to test *every single one* - I'll just do a few **free**, popular ones.

| Application   | Can Be Modified       |    
| :------------ | :-------------------: | 
| Discord       |           ✅          |                   
| Microsoft Teams (Classic)   | ✅        |                   
| VS Code*      |           ✅          |                   
| Obsidian      |           ✅          |                   
| Signal        |           ❌          |                                        
| Slack         |           ❌          |         

So after my *extensive* testing (of 6 programs), only Signal and Slack would not execute calc.exe, in fact they just didn't launch anything at all. Also Visual Studio Code is a bit of an interesting one, in that it doesn't have an ASAR archive and instead contains a folder called `app` which contains everything the `app.asar` archive would, meaning that you can perform the exploit without having to create an ASAR archive.

### Electron Security
So why did some Electron apps work and some didn't? While it may be for other reasons, my assumption is that Signal and Slack have implemented [ASAR Integrity](https://www.electronjs.org/docs/latest/tutorial/asar-integrity), an "experimental" feature introduced in Electron version 30.0.0 (on Windows, version 16.0.0 on Mac) which validates the contents of an Electron apps ASAR archive. Unfortunately, each Electron application uses a different version of Electron (which also means they're using their own versions of Chrome, each with their own vulnerabilities but thats a whole other story) - most are below the 30.0.0 requirement for ASAR Integrity. Furthermore, even apps that are above the version 30.0.0 requirement also need to enable the feature. Also considering the feature is considered "experimental" there are likely ways to bypass it.

### A Bright Idea
We can execute code in *most* Electron applications (a majority of which are reputable and trusted programs), so naturally I thought it would be a **great** idea to create a C2 server which runs within Electron applications. You're not going to have some random `pleasetrustme.exe` binary that can be analysed by an AV, no injecting shellcode into a running process, just a little bit of JavaScript tacked onto an `app.asar` file that nobody* is really checking, all the commands are being executed by a relatively trusted executable (even if they shouldn't be executing commands anyway).

So that's when the idea for Rogue Electron came about, essentially a JavaScript C2 that can be injected into an ASAR archive for execution.

## Rogue Electron
### Development
Originally, the Rogue Electron server was written in C, and was very terrible. In the initial stages you could execute commands and that was about it. It worked, but it was hard to maintain and infitely harder to add new features. I wanted to implement file upload and downloads, multiple session handling, command history, and lots more.

My friend [Aurillium](https://github.com/Aurillium) ported the code to Python for me, using Uvicorn and FastAPI to handle the web requests. Then off I went, adding tonnes of new features: downloading, uploading, command history, generating asar malicious files, session handling. 
### How does it work?
#### Injecting into ASAR files
The aptly named `asargen.py` will use `npx asar e` (e for extract) to first extract the contents of an ASAR archive, finds the "main" JavaScript... script?... which launches the original program, put that into the implant code so that the original program will still launch (so the victim doesn't raise any questions when their Discord just randomly stops opening), then modifies the `package.json` file to execute the implant, and finally packs it all up into an ASAR archive titled `app.asar`.

#### The Implant
The implant polls the server every 5 seconds, using HTTPS GET requests, looking to see if there are any commands to run. If yes, run them, if no, poll again in 5 seconds. The commands are run using the node "child_process" module:

`require('child_process').exec;`

Which "Spawns a shell and executes the `command` within that shell" - [NodeJS official docs](https://nodejs.org/api/child_process.html#child_processexeccommand-options-callback). For some reason, a lot of Electron applications have "Node Integration" enabled, which allows the implant to work and execute commands. 

To make it sealthier, the polling requests are made to look like legitimate traffic (inspired by [Sliver](https://github.com/BishopFox/sliver)), with the implant generating random GET requests for files such as `index.html` or `icon.svg`. POST requests act similarly, with the requests modified to look like typical POST requests.

#### The Server
The server is essentially a generic C2 server, it's just designed to work with the implant and respond to the specific requests it sends. As I mentioned previously, the server is written in Python, using Uvicorn and FastAPI to handle the web requests.

### Detection
#### Anti-Virus
Rogue Electron is *immune* to signature based detection. It has no executable, the only trace that it's on the system is that the ASAR archive of the impacted Electron program has been modified. No Anti-Virus solution, *that I know of,* is scanning ASAR archives (maybe they should). Behaviour-based/Machine-Learning AV products will likely be able to detect the C2 traffic, as well as the command execution (hopefully). But assuming something like this is being run on your standard user running Windows Defender, its not gonna catch it. Majority of my testing was conducted on a fully up-to-date (23H2) Windows 11 machine with defender fully enabled and it never made a peep.

#### Behaviour
Depending on the Electron application, it may be normal for it to be executing commands - but generally applications like Discord or MS Teams should not be running `whoami`, so Rogue Electron can definitely be detected there. Also the polling a random IP every 5 seconds is **extremely** suspicious.

#### ASAR Modification
The most sure way to detect this is to set auditing on the `app.asar` file and detect when it is modified by anything that isn't the main application (or it's updater). On Windows this is a matter of setting a system access control list (SACL) on the folder containing the ASAR file (typically 'resources'), and auditing file modification (ensuring that the SACL applies to the files contained within the folder). This will generate `EVID 4663` Windows Security logs, detailing the file that was accessed, what was done to the file, and the process that did it - all useful information for constructing effective SIEM rules to detect this behaviour. 

### Prevention
#### What can you do?
Prevention is difficult, most Electron apps run at a user level so resticting access to the ASAR archive would most likely result in the app not working. As a system administrator, the only thing you can really do is limit the number of Electron applications within your environment. For the Electron apps that are necessary for your environment, ensure they're kept up-to-date and monitor their activity closely. As for Rogue Electron itself, ensure you have a decent EDR solution that can detect and kill any C2 activity (based on their behaviour, do not rely on signature based detections). 

#### Down to the developers
Prevention largely relies on the developers using the Electron framework to secure their apps. This includes using the latest version of the Electron framework, enabling the ASAR Integrity feature, and disabling NodeJS (in new versions of Electron it is almost entirely uneeded). Some developers have even moved away from Electron (albeit, to something really not that far off), Microsoft ported Teams (Classic) to WebView2 (which is basically just Electron running on Edge instead of Chrome (which Edge runs on anyway ??)) and rebranded it to Teams (New) (look, don't even get me started on the MS Teams naming schemes).

## Conclusion 
At face value, Electron seems like a good idea. It makes cross-platform development **very** easy, and if you're already a web developer why learn a whole new programming language? That being said, the security concerns around Electron are well... concerning. And with Microsoft's introduction of WebView2, more and more apps are going to be built in similar ways and thus, have similar vulnerabilities. 

You can check out Rogue Electron on [GitHub](https://github.com/declangray/Rogue-Electron), and keep an eye out for updates as I'm still actively working on it!