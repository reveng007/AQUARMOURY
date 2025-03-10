# Goblin
[![](https://img.shields.io/badge/Category-Defense%20Evasion-E5A505?style=flat-square)]() [![](https://img.shields.io/badge/Language-C%20%2f%20C++%20%2f%20Python3-E5A505?style=flat-square)]()

## Introduction
`Goblin` is a module to enumerate all the threads of the `EventLog` Service Module(`wevtsvc.dll`) and kill them in an effort to disable `EventLog` service from registering any new events even though the service appears to be running. Disabling Windows Event Logging and Sysmon logging paves the way for operators to perform Post-Exploitation activities safely and stealthily.

![wevtsvc.dll Threads](https://github.com/reveng007/AQUARMOURY/blob/master/Goblin/Screenshots/evtlog-threads.PNG "wevtsvc.dll Threads")

Additionally, it also allows us to "revive" the `EventLog` service again without requiring a reboot after we are done with Post-Ex activities.

This tool was created to aid red team operators/penetration testers and to learn the inner workings of Windows Event Logging.

## Usage
### Using the pre-built binary
Grab the latest version from [here](https://github.com/slaeryan/AQUARMOURY/releases).
### Compiling yourself
Make sure you have a working VC++ 2019 dev environment set up beforehand and `Git` installed. Execute the following from an x64 Developer Command Prompt.
```
1. git clone https://github.com/slaeryan/AQUARMOURY
2. cd AQUARMOURY/Goblin
3. compile64.bat
```

`goblin_x64.dll` is the name of the module and it is quite small(about `7 kB` in size). It is converted to a PIC blob(shellcode) with the help of [sRDI](https://github.com/monoxgas/sRDI) courtesy of [@monoxgas](https://twitter.com/monoxgas?lang=en) and delivered straight to memory via your favourite C2 framework for inline execution/local execution in the implant process.

When the `Goblin` module is executed on a host, it primarily has one of these two objectives to accomplish:
1. Kill `wevtsvc.dll` threads if they are running
2. If `wevtsvc.dll` threads are not running, "revive" the `EventLog` service

Ergo, run the module once to disable event logging and once again if you wish to re-enable event logging without requiring a reboot.

In case your C2 framework does not support inline execution of shellcode for stability issues, you can also use the `Fork&Run` feature but the former is preferred over the latter due to OPSEC concerns.

Keep in mind that this capability is meant to be run from an **Elevated** context and will not work if the process token does not have `SeDebugPrivilege`.

## OPSEC concerns
This tool is inspired by the great [Invoke-Phant0m](https://github.com/hlldz/Invoke-Phant0m) by [Halil Dalabasmaz](https://twitter.com/hlldz).

So why not use PowerShell?

Because using PowerShell might not be the most OPSEC-safe way of doing this in 2020. There are numerous telling events generated by just running `Invoke-Phant0m` on a host from Enhanced PowerShell logging to Process Access Event(Sysmon Event ID 10).

Refer to [1](https://twitter.com/inzlain/status/867172350457925632/photo/1) and [2](https://malwarenailed.blogspot.com/2017/10/update-to-hunting-mimikatz-using-sysmon.html) for additional information on how to detect `Invoke-Phant0m`.

Enter Goblin.

![Goblin Overview](https://github.com/reveng007/AQUARMOURY/blob/master/Goblin/Screenshots/overview.PNG "Goblin Overview")

The first `notepad.exe` was started before running the `Goblin` module on the host and hence reported. The second one was launched after killing the `EventLog` service module threads and as expected the `notepad.exe` process creation event(Sysmon Event ID 1) never showed up in Sysmon logs. Note the time difference underlined in red, operators were successfully able to conduct Post-Ex activities during this time without any of it reported by Sysmon or being forwarded to SOC/SIEM.

After conducting Post-Exploitation we decided to enable logging again so as not to raise questions as to why a host has stopped reporting events altogether.

So we run the tool again to "revive" the `EventLog` service. What this does under the hood is kill the process hosting the service(`svchost.exe`) and start the service again.
As is evident from the screenshot, this generates at least **one** potentially telling event(command line parameter of `svchost.exe` containing string `EventLog`) which might give us away. 

Note that I said **one** because the other two events - The execution of the `loader.exe` and its termination(Sysmon Event ID 1 and 5) can be avoided in a live operation.

Key Takeaways:
1) **Running the `Goblin` module once\Killing the `EventLog` service threads** do not report any kind of additional event itself so it should be **OPSEC-safe** compared to **Running the `Goblin` module twice\Restarting the `EventLog` service** which is **noisy** and will report additional events providing Blue-teams with an opportunity to detect us.
2) While the `wevtsvc` threads are killed, the host will **not be reporting/forwarding any events** to SOC/SIEM even though the `EventLog` service is running which may or may not be noticed and alerted.
3) This will not stop PSPs relying on ETW Tracing to detect malicious activity. For that, we need to hook `NtTraceEvent` syscall from Kernel-mode(Ring-0).

For a more elegant solution that allows filtering of events reported, see [@bats3c](https://twitter.com/_batsec_) work on [EvtMute](https://github.com/bats3c/EvtMute).

## Detection
Here is a mandatory [CAPA](https://github.com/fireeye/capa) scan result of the `Goblin` DLL.
![CAPA Scan](https://github.com/reveng007/AQUARMOURY/blob/master/Goblin/Screenshots/capa.PNG "CAPA Scan")

And here is an additional `System` event reported as a result of "reviving" the `EventLog` service(Event ID 7031)
![Detection](https://github.com/reveng007/AQUARMOURY/blob/master/Goblin/Screenshots/detection.PNG "Detection")
Sometimes there also appears to be a `System` log indicating `EventLog` service has crashed(Event ID 7034).

Note that by killing the `EventLog` service threads, **NO** additional events show up in the event logs whatsoever. Detection from event logs is possible iff operator has restarted the service.

Another point to note is that enabling `EnableSvchostMitigationPolicy` enables `ACG` and `CIG` of `svchost.exe` which in turn makes running `EvtMute` non-trivial but would have no effect on this technique since it is not reliant on process injection and trampolines.
![svchost.exe Mitigation](https://github.com/reveng007/AQUARMOURY/blob/master/Goblin/Screenshots/svchost-mitigation.PNG "svchost.exe Mitigation")

## Credits
1. This tool was inspired by [@spotheplanet](https://twitter.com/spotheplanet) lab on [Disabling Windows Event Logs by Suspending EventLog Service Threads](https://www.ired.team/offensive-security/defense-evasion/disabling-windows-event-logs-by-suspending-eventlog-service-threads). Although, suspending/resuming threads do not work in practice because all the events are going to be written to the event Logs once the threads are resumed, it is an excellent post that explains in great detail the process of finding `wevtsvc.dll` threads. The code and algorithm are hacked from the post and I'd highly recommend giving it a read.
2. [https://artofpwn.com/phant0m-killing-windows-event-log.html](https://artofpwn.com/2017/06/05/phant0m-killing-windows-event-log.html)
3. [@dtm](https://twitter.com/0x00dtm) for first bringing this to my attention while discussing ways to evade Sysmon.
4. As usual, [@reenz0h](https://twitter.com/Sektor7Net) and [RTO: MalDev course](https://institute.sektor7.net/red-team-operator-malware-development-essentials) for the templates that I keep using to this date.
5. [@monoxgas](https://twitter.com/monoxgas?lang=en) for sRDI.
6. [@SBousseaden](https://twitter.com/sbousseaden) for the detection methodologies.
7. [@Nikhith_](https://twitter.com/Nikhith_) for taking the time to review the tool and the post.

## Author
Upayan ([@slaeryan](https://twitter.com/slaeryan)) [[slaeryan.github.io](https://slaeryan.github.io)]

## License
All the code included in this project is licensed under the terms of the GNU GPLv2 license.

#

[![](https://img.shields.io/badge/slaeryan.github.io-E5A505?style=flat-square)](https://slaeryan.github.io) [![](https://img.shields.io/badge/twitter-@slaeryan-00aced?style=flat-square&logo=twitter&logoColor=white)](https://twitter.com/slaeryan) [![](https://img.shields.io/badge/linkedin-@UpayanSaha-0084b4?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/upayan-saha-404881192/)
