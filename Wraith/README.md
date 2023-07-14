# Wraith
[![](https://img.shields.io/badge/Category-Defense%20Evasion-E5A505?style=flat-square)]() [![](https://img.shields.io/badge/Language-C%20%2f%20C++%20%2f%20CSharp%20%2f%20Python3-E5A505?style=flat-square)]()

## Introduction
Wraith is a native loader designed to pave the way for the arrival of a **Stage-1/Beaconing implant** or **Stage-2/Post-Ex implant** in-memory securely and stealthily. Specially designed to operate in heavily-monitored environments, it is designed with **PSP Evasion** as its primary goal.

## How To Use `Wraith`
Here is a guide to building `Wraith` in seven simple steps:
```
1) git clone https://github.com/slaeryan/AQUARMOURY.git & cd Wraith
2) Use Python/AES.py to encrypt the C2 Binary Payload/Shellcode and upload the payload to your staging server and the passphrase to the key server as text files
3) cd Src AND Open Config.h in your favourite text editor
4) Modify the configuration options to match your target
5) Use Python/StringMangler.py to encrypt the strings in the Config file AND Modifications to other parts of the Src is strictly not necessary
6) cd .. & compile64.bat
7) You'll find the compiled DLL, sRDI PIC blob and the ready-to-deploy sgn-encoded PIC blob in the Bin folder
```

A future version of `Wraith` will include a `Python` script to automate the process.

The loader itself is fairly small(`~40 kB`). It is compiled to a DLL and converted into a PIC blob using [sRDI](https://github.com/monoxgas/sRDI) courtesy of [@monoxgas](https://twitter.com/monoxgas?lang=en). It can be executed locally using a `Shellcode Execution Cradle`(Ex: in `C#` cradle for use with `D2JS/G2JS`) or executed directly in-memory via the `Stage-0` payload.

Here's an overview of the flow:

![Running Wraith](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/running-wraith.PNG "Running Wraith")

Read below to know what are some of the OPSEC concerns faced by existing toolings and how `Wraith` aims to solve some of them.

## OPSEC Concerns & how we can address them
### Network Activity Monitoring - `IE COM Object`

One of the ways EDRs can detect our tooling is by monitoring for processes that reach the internet especially:
1) If the context is "untrusted"
2) The context is "trusted" but does not load `Wininet.dll` or `Winhttp.dll` or is not expected to have any network activity, reaches for suspicious domains etc.

So it follows that if we go for **Remote Payload Fetching** then it has to be done from a context from where **network activity is not considered unusual so that we can blend in with legitimate traffic**.
Ex: Firefox, Chrome, Edge, IE, other browsers etc.

Luckily for us attackers, we can automate an IE browser instance programmatically using [IE COM Object](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/aa752084(v=vs.85)) in the background without any visible windows.

![IE COM](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/ie-com.PNG "IE COM")

This is a screenshot demonstrating how we can fetch arbitrary text data from our `Payload Staging Server`(Here seen using `AWS S3 bucket`) with `IE COM Object`.

Pretty neat eh? :)

One thing I want to point out here is that we will leave a crucial bread crumb this way for Defenders to follow. Can you guess what it is?

Yes, it is the IE Browser History which would reveal the payload URL among other things.

Luckily for us attackers(yet again!), MS provides with an interface known as [IUrlHistoryStg2](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/ms774948(v=vs.85)) which has a method called [ClearHistory](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/ms774947(v=vs.85)) to delete all browsing history data for the current user.

This way we can ensure that we leave absolutely no traces behind ;)

### Execution on Non-targeted asset/Sandbox - `Execution Guardrails`
This is done to:
1) To prevent the accidental breaking of the rules of engagement. This will ensure that our malcode doesn’t end being executed on any unintended host which are out of the scope
2) To protect IP and hinder the efforts of blue teams trying to run/reverse engineer the implant on non-targeted assets and thwart analysis on automated malware sandboxes, AV/EDR emulators etc.

To this effect we have implemented 4 safety checks in our loader:
1) **Mutex Check - To enforce Single-Execution of payload**. This is done by attempting to create a mutex when the loader is executed and checking whether it already exists in which case the loader will terminate immediately. Otherwise, carry on execution as intended. This is obviously necessary since we don't want to execute the payload more than once on the same host whether accidentally or intentionally.
2) **Kill Date - To render the loader(and by extension the payload) harmless after the engagement ends**. This is done by checking the current date on the host with the hard-coded kill date and if it exceeds the kill date then we simply do not proceed.
3) **Static Endpoint Validation - To ensure our payload doesn't run on non-targeted assets**. This is done by comparing a `SHA-256` hash of host artefact(Host Name/Domain Name) retrieved at runtime with the hard-coded `SHA-256` hash value of the same. If it matches, assume that our code has detonated on a targeted asset and proceed execution otherwise terminate. Additionally, we hard-code the hashed value instead of the plaintext value itself to make the process of identifying the intended target non-trivial.

![Workstation Artifact](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/workstation.PNG "Workstation Artifact")

![Domain-Joined Artifact](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/domain-joined.PNG "Domain-Joined Artifact")

Here we can see the respective artefact name and values for `Workstation` and `Domain-Joined` machines respectively.

4) **Dynamic Endpoint Validation - To have an emergency payload kill-switch**. This is done by encrypting the payload using `AES-256` in `CBC mode` with a `random IV` and a passphrase that is stored on a remote `Key Server`(again fetched using `IE COM Object`) instead of hard-coding it in the loader itself. This way if we suspect that we might be compromised in any way or blue teams are onto us, we can simply delete the passphrase from the keyserver or replace it with an incorrect one which in turn would make payload decryption and ergo execution fail.

All these elaborate safety measures also help us to protect our IP. In a worst-case scenario, our `Payload Staging Server` and `Key Server` are burned but we can still save our later stages - C2 infrastructure and payload. This also partly explains our motivation behind using a loader to deliver the C2 payload.

Here is a screenshot highlighting the flow:

![Execution Guardrails](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/execution-guardrail.PNG "Execution Guardrails")

And here's how it looks when executed on a non-targeted asset:

![Non Targeted](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/non-targeted.PNG "Non Targeted")

### Detection of Injection/C2 payload - `"Advanced Bird" APC Injection`
Part of the motivation behind using a loader is **to deliver the payload into the address space of a legitimate, signed and trusted process from where the beaconing network activity is not going to be flagged.** One of the ways to achieve this is using Process Injection.

We have chosen to use a variant of the [Early Bird APC Injection](https://www.ired.team/offensive-security/code-injection-process-injection/early-bird-apc-queue-code-injection) technique(playfully dubbed "Advanced-Bird" lol!) because we prioritise a more functional and stable technique over some exotic ROP-based injection containing over 200+ lines of code which is eventually going to get burned :)

This technique in its base form relies on spawning a "trusted" sacrificial process in a suspended state, allocating memory/writing the payload to the target process and finally queuing an APC routine to the primary suspended thread pointing to the shellcode before resuming the thread to execute our malcode.

Even though Sysmon is going to report a **Process Creation Event(Sysmon Event ID 1)**, it has some potential benefits:
1) **We do not rely on `OpenProcess` to get a handle to an external process which is going to be detected by tools using `ObRegisterCallbacks`**
2) **We can fully take the advantage of mitigation policies such as `CIG/ACG` to protect our injected C2 payload**

And needless to say, Sysmon currently cannot detect APC process injection(MDATP however can via `THREATINT_QUEUEUSERAPC_REMOTE_KERNEL_CALLER`!).

I have taken the liberty to change a few things in the original algorithm to aid in evasion. Ex: Removed the creation of a suspended process which can be a bit of an **IoC**. **The workaround first creates a process the usual way and then suspends the primary thread**.

But using such a well-known technique comes with a potential problem. It uses some pretty _cursed_ API calls in a particular sequence which are going to be picked up by any half-decent AV/EDR.

[Direct Syscalls](https://outflank.nl/blog/2019/06/19/red-team-tactics-combining-direct-system-calls-and-srdi-to-bypass-av-edr/) to the rescue!

Next question is to hardcode or not?

We have chosen to use hardcoded syscall stubs for our loader thanks to [@j00ru](https://twitter.com/j00ru?lang=en) [syscall table](https://j00ru.vexillium.org/syscalls/nt/64/). However, considering the fact that syscall numbers change between different versions of the OS and sometimes even between different Windows 10 builds, this is probably not a good choice(with DevOps people already flustered :)) The other alternative is extracting the syscall stub from `Ntdll` at runtime to free us of the requirement of hardcoding them. However, in my opinion, this is a trade-off between stealth vs. scalability with the former being slightly stealthier than the latter especially during RE of the implant(Ex: reading `Ntdll` from disk, looping through Export Table etc.). For red-team engagements with thorough reconnaissance of the target, hardcoding syscall stubs typically shouldn't be a problem though.

There is a nice implementation of dynamic syscalls known as [HellsGate](https://github.com/am0nsec/HellsGate) by [@am0nsec](https://twitter.com/am0nsec?lang=en) and [@smelly_vx](https://twitter.com/smelly__vx) but in the present form, EDR trampolines potentially render it useless :( 

Another viable alternative created by [@modexp](https://twitter.com/modexpblog) uses [Exception Directory to read the syscall stub](https://modexp.wordpress.com/2020/06/01/syscalls-disassembler/).

This is a screenshot of using "normal" APC Injection without syscalls which clearly shows that our API calls were intercepted by our pseudo-EDR a.k.a. [API Monitor](http://www.rohitab.com/apimonitor):

![API Monitor Detected](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/api-monitor-detected.PNG "API Monitor Detected")

And this shows how our injection technique with direct syscalls would look through the lens of an EDR:

![API Monitor Undetected](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/api-monitor-undetected.PNG "API Monitor Undetected")

As we can see, our injection wasn't intercepted by our _EDR_ and the **assumption here being that the EDR relies on Ring-3/User-Mode hooks instead of KernelMode ETW Threat Intelligence functions to gain visibility into potentially suspicious actions**(which most of them do not thanks to MS :))

Oh, and if you're wondering about the `NtAllocateVirtualMemory/NtWriteVirtualMemory` calls, it is actually called internally by `CreateProcessA` as visible from the Call Stack and the ones at the beginning highlighted in red are related to the shellcode execution cradle which inline-executes our injector blob and does not pertain to our remote injection technique itself as visible from the first argument which is `GetCurrentProcess`.

So now that we have managed to hide the act of injection itself, how do we protect our C2 payload which is proprietary and the source unavailable for modification?

Enter [CIG and ACG](https://blog.xpnsec.com/protecting-your-malware/) by [Adam Chester a.k.a. @_xpn_](https://twitter.com/_xpn_)!

`CIG` also popularly known as `blockdlls` is an MS mitigation policy that **prevents any non-MS signed third-party DLL(Ex: EDR Hooking DLL) from being injected into our spawned sacrificial "trusted" process which now contains the C2 payload**. 

But some EDRs were quick to adapt to this mitigation policy and they quickly got their _evil_ DLL signed by MS which would render this useless. Bummer :(

Fortunately for us, there exists another mitigation policy known as `ACG` or `Arbitrary Code Guard` which **prevents a process's ability to allocate new executable pages and/or change existing executable page protections which is required for EDR trampolines to work**. This means that EDR _evil_ DLL injection would fail even if it were signed ;)

The caveat here is that [Cobalt Strike Beacon payload](https://www.cobaltstrike.com/help-beacon) is currently **NOT** compatible with `ACG` and would break it. So readers be forewarned!

`Wraith` supports both `CIG` and `ACG` and can be configured using the config file.

One thing I want to point out here is that I have seen some blogs and tools using `SetMitigationPolicy` to add `ACG` and `CIG` after a process is created. I think that this is a flawed approach and it would not stop an EDR DLL getting injected from the kernel into the process. The idea here is to create a process **with the mitigations enabled**.

Now would be a good time to introduce [PPID Spoofing](https://blog.didierstevens.com/2009/11/22/quickpost-selectmyparent-or-playing-with-the-windows-process-tree/) into the conversation which is **used to break the process chain(Word/Excel spawning our sacrificial process) by choosing an appropriate parent-child pair(once again configurable using config file) to blend in and protect our payload from getting caught**.

Here's how it looks in-action:

![Protecting C2 Payload](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/wraith-acg-ppid.PNG "Protecting C2 Payload")

In this way we can counter the holy trio of EDR detection using:

1) **API hooking - `Direct Syscalls + CIG/ACG`**
2) **Abnormal parent-child process relationship and command-line arguments - `PPID Spoofing + Cmdline Spoofing`**
3) **Logging network activity of processes - `Injection into a legitimate process to blend in with legitimate traffic`**

Now, of course, nothing is foolproof and just as detection isn't flawless so are the countermeasures for these detections.

For example: `ACG` doesn't stop a remote process's ability to allocate/modify executable pages using `VirtualAllocEx/VirtualProtectEx`. Also, it could be [turned off](http://blog.sevagas.com/IMG/pdf/code_injection_series_part4.pdf) rather easily but EDRs usually stay away from any kinds of bypasses so that could potentially work in our favour :)

`PPID Spoofing` can be detected too using `ETW`(More on this later in the `Detections` section).

### Memory Analysis - `Thread Execution Hijacking`
To hinder memory analysis, we have chosen to use a variant of `APC Injection` which **hijacks the execution flow of the primary thread of our sacrificial process instead of creating a remote thread in the target process which is unbacked by a module on disk ergo trivial to detect** using tools such as [Get-InjectedThread](https://gist.github.com/jaredcatkinson/23905d34537ce4b5b1818c3e6405c1d2).

This can be verified using [PE-Sieve](https://github.com/hasherezade/pe-sieve) too:

![PE Sieve](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/pesieve.PNG "PE Sieve")

Another alternative is using [SiR Injection]() i.e. to hijack thread execution by redirecting `RIP` register to our payload using `GetThreadContext/SetThreadContext`.

Furthermore, we compile our loader DLL with a custom MS-DOS stub(with `/STUB` linker flag) that does not contain the message `This program can not be run in DOS mode`. Not that it would matter anyway because we delete the PE header of the loader PIC blob after reflective loading using the `-c` flag of `sRDI`.

Quoting from the [sRDI](https://github.com/monoxgas/sRDI) project:
```
SRDI_CLEARHEADER [0x1]: The DOS Header and DOS Stub for the target DLL are completley wiped with null bytes on load (Except for e_lfanew). This might cause issues with stock windows APIs when supplying the base address as a psuedo HMODULE.
```

Now, this wouldn't do anything for the injected payload. Some possible improvements could include the integration of [Gargoyle](https://github.com/JLospinoso/gargoyle) i.e. using ROP activators to hide in non-executable memory in periodic intervals or when a memory scan is triggered due to some suspicious API calls. Also, the injection technique itself could be improved by using something like [AEP Injection]() i.e. To overwrite the image entry point of our sacrificial process with our payload. 

### Static Detection - `Polymorphic Encoder + Sensitive String Obfuscation + Dynamic API Resolving`
To hinder Blue Teams from signaturing our loader PIC blob, I have taken the liberty to polymorphic encode the loader shellcode with [sgn](https://github.com/EgeBalci/sgn) courtesy of [Ege Balci](https://twitter.com/egeblc).

Here's how a "stock" shellcode looks on VT:

![Unencoded Shellcode](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/unencoded-shellcode.png "Unencoded Shellcode")

And here's how the same shellcode looks after a single pass from a polymorphic encoder:

![Encoded Shellcode](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/encoded-shellcode.png "Encoded Shellcode")

Furthermore, we dynamically resolve most of the _suspicious_ API functions at runtime using `LoadLibraryA/GetModuleHandleA + GetProcAddress` to achieve a clean import table.

Apart from this, we also obfuscate/encrypt sensitive strings in the loader with a single byte `XOR` key to prevent blue teams from running something like `strings` or [floss](https://github.com/fireeye/flare-floss) on our shellcode to obtain sensitive strings.
Ex: The `Payload Staging Server` URL, `Key Server` URL, Hashed artefact value, Mutex Name and so on.

## Detections
This post would be incomplete without briefly mentioning some of the ways by which we can detect our tooling.

Here is a mandatory [CAPA](https://github.com/fireeye/capa) scan result of the loader shellcode:

![CAPA](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/capa.PNG "CAPA")

Note that it provides us with almost no intel since we have a clean import table.

And here is the Sysmon log using [SwiftOnSecurity's Sysmon config](https://github.com/SwiftOnSecurity/sysmon-config) of running `Wraith` on the host:

![Sysmon](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/wraith-sysmon.PNG "Sysmon")

The red `+` denotes that these two events(Sysmon Event ID 1 & 5) are not a part of the loader blob and can be avoided in a real-life operation. The rest of the events reported are quite obvious as one might expect. There's a Process Creation Event of the sacrificial process, another process creation event of the `IE Browser` and a DNS Query Event(Sysmon Event ID 22) fired from the browser.

Note that if we enabled Process Access Event(Sysmon Event ID 10), we would have gotten at least one handle access for obtaining a handle to the parent process for `PPID Spoofing`. 

Here is a [YARA](https://github.com/VirusTotal/yara) rule to detect hardcoded direct syscalls courtesy of [Samir Bousseaden a.k.a @SBousseaden](https://twitter.com/sbousseaden):

![YARA](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/yara.PNG "YARA")

The rule is included in the repo.

Here's a screenshot showing detection of `PPID Spoofing`:

![PPIDetector](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/detect-ppid-spoofing.PNG "PPIDetector")

Remember how we said that `PPID Spoofing` [can be detected](https://www.ired.team/offensive-security/defense-evasion/parent-process-id-ppid-spoofing)? This is done by creating a trace session using `Microsoft-Windows-Kernel-Process` as an ETW provider and correlating between `ExecutionProcessID` and `ParentProcessID` field.

I have included the built and `ILMerge`'d PoC to detect `PPID Spoofing` in the repo.

Lastly, here is a memory sweep of the payload process/sacrificial process with [Moneta](https://github.com/forrest-orr/moneta):

![Moneta](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/moneta.PNG "Moneta")

It does provide us with an alert saying `Abnormal private RX memory` which may not always necessarily indicate that something's wrong but it definitely warrants a second look and in this case we inspect further with ProcessHacker to confirm that the memory flagged by `Moneta` does indeed contain our payload(`MessageBox` in our case).

## EDIT
So some people have asked me why I had to get a pointer to `PEB` struct via `TEB` in [line 340](https://github.com/reveng007/AQUARMOURY/blob/1923e65190875f7c61c76fb430d526e5deaa062a/Wraith/Src/Syscalls.h#L340) when I could have retrieved it directly like:
```
PPEB peb = (PPEB)__readgsqword(0x60)
```

Furthermore, if we need `PEB` only to get OS version for syscalls why even bother with PEB instead of calling `RtlGetVersion`?

The answer is this:

![PEB Access](https://github.com/reveng007/AQUARMOURY/blob/master/Wraith/Screenshots/peb-access.png "PEB Access")

As highlighted by the above screenshot, `CAPA` was detecting my `PEB` access and I did not want to ruin my clean result :)

I could have used something like `NtQueryInformationProcess` to get pointer to `PEB` like this:
```
	NTSTATUS status = NtQueryInformationProcess(hProc, ProcessBasicInformation, &pbi, sizeof(pbi), 0);
	if (status == STATUS_SUCCESS) {
		return pbi.PebBaseAddress;
	}
```

But on the off chance that the function is hooked, I decided to get PEB via TEB. Same goes for `RtlGetVersion`(and not to mention, it seemed easier than the above-mentioned alternatives for someone as lazy as me :))

Also, if we have `PEB` defined then why not use it to resolve imports and get rid of `GetModuleHandleA + GetProcAddress`?

Well, the answer is that we aren't directly writing PIC in C and lots of legitimate software use the above to resolve API functions at run-time so it really wasn't a concern. But with that being said, it'd probably be a nice idea to incorporate it anyway and I'm accepting PRs :)

I always appreciate constructive criticism/feedback and improvement suggestions so shout out to [@smelly_vx](https://twitter.com/smelly__vx) for the feedback!

## TO-DO
- [ ] Add support for command-line spoofing for the sacrificial process
- [ ] Create a Python helper script to automate the build process
- [ ] Integrate AEP injection as an alternative technique
- [ ] Add error handling + some minor code cleanup
- [ ] Add support for Dynamic Syscalls
- [ ] Add support for PEB import hiding + x64 Gargoyle

More to be added soon...

## Credits
1. [@_xpn_](https://twitter.com/_xpn_) for introducing `ACG/CIG` for implant safety
2. [@MWRLabs](https://twitter.com/mwrinfosecurity?lang=en) for [Safer Shellcode Implants](https://labs.f-secure.com/archive/safer-shellcode-implants/)
3. [@spotheplanet](https://twitter.com/spotheplanet) and [ired.team](https://www.ired.team/) for interesting tidbits used throughout the post
4. [@OutflankNL](https://twitter.com/outflanknl?lang=en) for [https://outflank.nl/blog/2019/06/19/red-team-tactics-combining-direct-system-calls-and-srdi-to-bypass-av-edr/](https://outflank.nl/blog/2019/06/19/red-team-tactics-combining-direct-system-calls-and-srdi-to-bypass-av-edr/)
5. [@dtm](https://twitter.com/0x00dtm) for the wonderful OPSEC discussions on [0x00sec VIP](https://discord.com/invite/c6BHVfn)
6. As usual, [@reenz0h](https://twitter.com/Sektor7Net) and [RTO: MalDev course](https://institute.sektor7.net/red-team-operator-malware-development-essentials) for the templates that I keep using to this date
7. [@monoxgas](https://twitter.com/monoxgas?lang=en) for sRDI
8. [@SBousseaden](https://twitter.com/sbousseaden) for the detection methodologies

## Author
Upayan ([@slaeryan](https://twitter.com/slaeryan)) [[slaeryan.github.io](https://slaeryan.github.io)]

## License
All the code included in this project is licensed under the terms of the GNU GPLv2 license.

#

[![](https://img.shields.io/badge/slaeryan.github.io-E5A505?style=flat-square)](https://slaeryan.github.io) [![](https://img.shields.io/badge/twitter-@slaeryan-00aced?style=flat-square&logo=twitter&logoColor=white)](https://twitter.com/slaeryan) [![](https://img.shields.io/badge/linkedin-@UpayanSaha-0084b4?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/upayan-saha-404881192/)
