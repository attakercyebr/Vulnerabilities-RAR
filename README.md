Introduction
In this article, we tell the story of how we found a logical bug using the WinAFL fuzzer and exploited it in WinRAR to gain full control over a victim’s computer. The exploit works by just extracting an archive, and puts over 500 million users at risk. This vulnerability has existed for over 19 years(!) and forced WinRAR to completely drop support for the vulnerable format.

Background
A few months ago, our team built a multi-processor fuzzing lab and started to fuzz binaries for Windows environments using the WinAFL fuzzer. After the good results we got from our Adobe Research, we decided to expand our fuzzing efforts and started to fuzz WinRAR too.

One of the crashes produced by the fuzzer led us to an old, dated dynamic link library (dll) that was compiled back in 2006 without a protection mechanism (like ASLR, DEP, etc.) and is used by WinRAR.

We turned our focus and fuzzer to this “low hanging fruit” dll, and looked for a memory corruption bug that would hopefully lead to Remote Code Execution.
However, the fuzzer produced a test case with “weird” behavior. After researching this behavior, we found a logical bug: Absolute Path Traversal. From this point on it was simple to leverage this vulnerability to a remote code execution.

Perhaps it’s also worth mentioning that a substantial amount of money in various bug bounty programs is offered for these types of vulnerabilities.

The Fuzzing Process Background
These are the steps taken to start fuzzing WinRAR:

Creation of an internal harness inside the WinRAR main function which enables us to fuzz any archive type, without stitching a specific harness for each format. This is done by patching the WinRAR executable.
Eliminate GUI elements such as message boxes and dialogs which require user interaction. This is also done by patching the WinRAR executable.
There are some message boxes that pop up even in CLI mode of WinRAR.
Use a giant corpus from an interesting piece of research conducted around 2005 by the University of Oulu.
Fuzz the program with WinAFL using WinRAR command line switches. These force WinRAR to parse the “broken archive” and also set default passwords (“-p” for password and “-kb” for keep broken extracted files). We found those options in a WinRAR manual/help file.
After a short time of fuzzing, we found several crashes in the extraction of several archive formats such as RAR, LZH and ACE that were caused by a memory corruption vulnerability such as Out-of-Bounds Write. The exploitation of these vulnerabilities, though, is not trivial because the primitives supplied limited control over the overwritten buffer.

However, a crash related to the parsing of the ACE format caught our eye. We found that WinRAR uses a dll named unacev2.dll for parsing ACE archives. A quick look at this dll revealed that it’s an old dated dll compiled in 2006 without a protection mechanism. In the end, it turned out that we didn’t even need to bypass them.

Build a Specific Harness
We decided to focus on this dll because it looked like it would be quick and easy to exploit.

Also, as far as WinRAR is concerned, as long as the archive file has a .rar extension, it would handle it according to the file’s magic bytes, in our case – the ACE format.

To improve the fuzzer performance, and to increase the coverage only on the relevant dll, we created a specific harness for unacev2.dll .

To do that, we need to understand how unacev2.dll is used. After reverse engineering the code calling unacev2.dll for ACE archive extraction, we found that two exported functions should be called for extraction in the following order:

An initialization function named ACEInitDll, with the following signature:
INT __stdcall ACEInitDll(unknown_struct_1 *struct_1);
• struct_1: pointer to an unknown struct
An extraction function named ACEExtract , with the following signature:
INT __stdcall ACEExtract(LPSTR ArchiveName, unknown_struct_2 *struct_2);
•ArchiveName: string pointer to the path to the ace file to be extracted
•struct_2: pointer to an unknown struct
Both of these functions required structs that are unknown to us. We had two options to try to understand the unknown struct: reversing and debugging WinRAR, or trying to find an open source project that uses those structs.

The first option is more time consuming, so we opted to try the second one. We searched github.com for the exported function ACEInitDll
and found a project named FarManager that uses this dll and includes a detailed header file for the unknown structs.
Note: The creator of this project is also the creator of WinRAR.

After loading the header files to IDA, it was much easier to understand the previously “unknown structs” to both functions (ACEInitDll and ACEExtract ),  as IDA displayed the correct name and type for each struct member.

From the headers we found in the FarManager project, we came up with the following signature:

INT __stdcall ACEInitDll(pACEInitDllStruc DllData);

INT __stdcall ACEExtract(LPSTR ArchiveName, pACEExtractStruc Extract);

To mimic the way that WinRAR uses unacev2.dll , we assigned the same struct member just as WinRAR did.

We started to fuzz this specific harness, but we didn’t find new crashes and the coverage did not expand in the first few hours of the fuzzing. We tried to understand the reason for this limitation.

We started by looking for information about the ACE archive format.

Understanding the ACE Format
We didn’t find a RFC for that format, but we did find vital information over the internet.

1. Creating an ACE archive is protected by a patent. The only software that is allowed to create an ACE archive is WinACE. The last version of this program was compiled in November 2007. The company’s website has been down since August 2017. However, extracting an ACE archive is not protected by a patent.

2. A pure Python project named acefile is mentioned in this Wikipedia page. Its most useful features are:

It can extract an ACE archive.
It contains a brief explanation about the ACE file format.
It has a very helpful feature that prints the file format header with an explanation.
To understand the ACE file format, let’s create a simple .txt file (named “simple_file.txt”), and compress it using WinACE. We will then check the headers of the ACE file using acefile .

Notes:

Consider each “\\” from the filename field in the image above as a single slash “\”, this is just python escaping.
For clarity, the same fields are marked with the same color in the hex dump and in the output fromacefile.
Summary of the important fields:

hdr_crc (marked in pink):
Two CRC fields are present in 2 headers. If the CRC doesn’t match the data, the extraction
is interrupted. This is the reason why the fuzzer didn’t find more paths (expand its coverage).To “solve” this issue we patched all the CRC* checks in unacev2.dll .*Note – The CRC is a modified implementation of the regular CRC-32.
filename (marked in green):
It contains the relative path to the file. All the directories specified in the relative path are created during the extracting process (including the file). The size of the filename is defined by 2 bytes (little endian) marked by a black frame in the hex dump.
advert (marked in yellow)
The advert field is automatically added by WinACE, during the creation of an ACE archive, if the archive is created using an unregistered version of WinACE.
file content:
“origsize ” – The content’s size. The content itself is positioned after the header that defines the file (“hdr_type” field == 1).
“hdr_size ” – The header size. Marked by a gray frame in the hex dump.
At offset 70 (0x46) from the second header, we can find our file content: “Hello From Check Point!”
Because the filename field contains the relative path to the file, we did some manual modification attempts to the field to see if it is vulnerable to “Path Traversal.”
For example, we added the trivial path traversal gadget “\..\” to the filename field and more complex “Path Traversal” tricks as well, but without success.

After patching all the structure checks, such as the CRC validation, we once again activated our fuzzer. After a short time of fuzzing, we entered the main fuzzing directory and found something odd. But let’s first describe our fuzzing machine for some necessary background.

The Fuzzing Machine
To increase the fuzzer performance and to prevent an I\O bottleneck, we used a RAM disk drive that uses the ImDisk toolkit on the fuzzing machine.



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
