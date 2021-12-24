# ![Locations](https://github.com/M4nifest0/M4nifest0_WhatsApp/blob/master/s.png) 

##### Program Features
----------------------
- 📌 Hide malware
- 📌 Vulnerability software vulnerabilities RAR
- 📌 Creating malware
- 📌 Access client files
- 📌 Client Hacking



##### Introduction
----------------------
- In this article, we tell the story of how we found a logical bug using the WinAFL fuzzer and exploited it in WinRAR to gain full control over a victim’s computer. The exploit works by just extracting an archive, and puts over 500 million users at risk.

- One of the crashes produced by the fuzzer led us to an old, dated dynamic link library (dll) that was compiled back in 2006 without a protection mechanism (like ASLR, DEP, etc.) and is used by WinRAR.

- We turned our focus and fuzzer to this “low hanging fruit” dll, and looked for a memory corruption bug that would hopefully lead to Remote Code Execution.

- However, the fuzzer produced a test case with “weird” behavior. After researching this behavior, we found a logical bug: Absolute Path Traversal. From this point on it was simple to leverage this vulnerability to a remote code execution.

- Perhaps it’s also worth mentioning that a substantial amount of money in various bug bounty programs is offered for these types of vulnerabilities.

# See how it work 
----------------------
- 🤡  

# Visit the following channels and sites for more training and tools:
----------------------
- 🔞 https://m4nifest0.com
- 🔞 https://m4nifest0.group
- 🔞 https://m4nifest0.shop
- 🔞 https://t.me/M4nifest0

----------------------

<h2>
<p align="center">	
</a>&nbsp;&nbsp;&nbsp;&nbsp;
	<a href="https://t.me/M4nifest0">
		<img src="https://img.shields.io/badge/Telegram-%23000000.svg?&style=for-the-badge&logo=Telegram&logoColor=white" />
	</a>&nbsp;&nbsp;&nbsp;&nbsp;
	<a href="https://twitter.com/_M4nifest0_">
		<img src="https://img.shields.io/badge/twitter-%231DA1F2.svg?&style=for-the-badge&logo=twitter&logoColor=white" />
	</a>&nbsp;&nbsp;&nbsp;&nbsp;
	<a href="https://m4nifest0.com">
		<img src="https://img.shields.io/badge/WebSite-%234A154B.svg?&style=for-the-badge&logo=slack&logoColor=white" />
	</a>&nbsp;&nbsp;&nbsp;&nbsp;
</p>
