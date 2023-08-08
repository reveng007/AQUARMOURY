# Gnome
[![](https://img.shields.io/badge/Category-Defense%20Evasion-E5A505?style=flat-square)]() [![](https://img.shields.io/badge/Language-C%20%2f%20C++%20%2f%20Python3-E5A505?style=flat-square)]()

## Introduction
`Gnome` is a module to load your signed driver **stealthily**. **The driver is extracted from the `Gnome` loader, dropped to disk and loaded using `NtLoadDriver` instead of the usual service creation driver loading which can be noisy and leaves large forensic artefacts behind such as service creation, service start/stop logs etc**.

It can be used to **drop'n'load your signed rootkit** in the target environment. It can also be used to load a vulnerable signed driver to execute **arbitrary Ring-0 code for privilege escalation/disabling PPL/disabling DSE/disabling EDR callbacks etc**.

While this tool may not directly aid in Defence Evasion, think of it as a very smol utility to aid in the events leading up to the bypassing of defences :)

## Usage
Here is a guide to building the tool in easy steps:
```
1. git clone https://github.com/slaeryan/AQUARMOURY.git & cd Gnome
2. Replace your driver that you want to be loaded in Bin dir as payload_x64.sys so that we can find it while building Gnome (There's already a self-signed test driver in the folder that just prints a couple of debug messages captured via DebugView)
3. Open Src/dllmain.cpp in a text editor and configure the driver name and path if necessary
4. From an x64 Developer Command Prompt, execute the batch script - compile64.bat
5. You'll find the compiled DLL, ready-to-deploy PIC blob in the Bin folder
```

As with most of the tools in this tool suite, `Gnome` is compiled to a DLL and converted to a PIC blob with the help of [sRDI](https://github.com/monoxgas/sRDI) courtesy of [@monoxgas](https://twitter.com/monoxgas?lang=en) and delivered straight to memory via your favourite C2 framework for inline execution/local execution in the implant process. It can also be injected to a remote process using `shinject` or `shspawn` (fork'n'run).

![Running Gnome](https://github.com/reveng007/AQUARMOURY/blob/master/Gnome/Screenshots/running-gnome.PNG "Running Gnome")

**On the first run, it will attempt to drop'n'load the payload driver and on the second run, it will unload the driver and clean up after itself**.

Keep in mind that this capability is meant to be run from an **Elevated** context and will not work if the process token does not have `SeLoadDriverPrivilege`.

Read below to know more about the module.

## Algorithm
As with all my other writeups, I tend to include as little code as possible in the README to make it more accessible and palatable for the masses.

So here goes the algorithm of the code that should help understand what goes on under the hood without having to wade through my shitty code (lol! although brownie points for doing so).

When `Gnome` is executed in-memory, the flow of its operation is quite simple and it goes something like this:
1. The payload driver that is embedded in the resource section of the `Gnome` module is extracted and stored in-memory before it is written to disk in the pre-specified location.
2. Next step is enabling the privilege in the access token that is required to load drivers known as `SeLoadDriverPrivilege/SE_LOAD_DRIVER_NAME`. It should be worth noting that this privilege is not available in a medium-IL process token to be enabled in the first place implying medium-IL processes shouldn't normally be able to load drivers which of course makes sense right?
3. Our next step is creating a couple of registry keys(and subkeys) and setting their respective values which are required for loading drivers via this technique. First, we create a key at `HKLM\SYSTEM\CurrentControlSet\Services\<driver name>` and four subkeys called `ImagePath`, `Type`, `Start` and `ErrorControl` under it. Then the values of the subkeys are set to driver path on disk, 1 (this is a kernel driver), 3 (we want a manual load) and finally 0 (since we do not load driver at startup) respectively.
Here's a screenshot that might help visualise this:

![Registry Entries](https://github.com/reveng007/AQUARMOURY/blob/master/Gnome/Screenshots/reg-entries.PNG "Registry Entries")

4. Now, we need to convert the registry service key path of our driver to `Unicode`. This is essential because `NtLoadDriver` requires to be fed a `Unicode` string. This can be achieved via `RtlInitUnicodeString`. Finally, we may call `NtLoadDriver` with the `Unicode` registry service path as the single argument to load the driver. Keep in mind that:
```
if (driver == MS-signed || driver == cross-signed) DSE happy and will load driver :)
if (driver != MS-signed || driver != cross-signed) DSE unhappy and will block driver loading :(
```
That is if Test Mode is not turned on like so:
```
1. Open CMD with administrative privileges
2. Type: bcdedit -set TESTSIGNING ON and hit enter
3. Reboot computer.
```
5. In case the loading fails, we assume that it failed because it was already loaded so we proceed to unload the driver using `NtUnloadDriver`, delete the dropped driver from disk, then delete the registry keys and subkeys we created in step 3 and finally exit. Theoretically, this logic is potentially flawed and it is a pretty bold assumption considering driver loading may fail due to a myriad number of reasons but practically all scenarios considered, it didn't appear to me as too much of a problem anyways.

## Detections
Here is a mandatory [CAPA](https://github.com/fireeye/capa) scan result of the `Gnome` DLL:

![CAPA](https://github.com/reveng007/AQUARMOURY/blob/master/Gnome/Screenshots/capa.PNG "CAPA")

And here is a screenshot showing Sysmon logs of running Gnome:

![Sysmon](https://github.com/reveng007/AQUARMOURY/blob/master/Gnome/Screenshots/sysmon.PNG "Sysmon")

As expected, there are a couple of `Sysmon Event ID 13`'s denoting registry value modifications as made while preparing to load the driver and finally the big event - `Sysmon Event ID 6` denoting that a driver was loaded successfully.

One of the detection strategies could include monitoring for these events for known vulnerable drivers such as the GigaByte driver shown in the above screenshot.

## Credits
1. iceb0y for [https://github.com/iceb0y/ntdrvldr](https://github.com/iceb0y/ntdrvldr)
2. [@monoxgas](https://twitter.com/monoxgas?lang=en) for [sRDI](https://github.com/monoxgas/sRDI)

## Author
Upayan ([@slaeryan](https://twitter.com/slaeryan)) [[slaeryan.github.io](https://slaeryan.github.io)]

## License
All the code included in this project is licensed under the terms of the GNU GPLv2 license.

#

[![](https://img.shields.io/badge/slaeryan.github.io-E5A505?style=flat-square)](https://slaeryan.github.io) [![](https://img.shields.io/badge/twitter-@slaeryan-00aced?style=flat-square&logo=twitter&logoColor=white)](https://twitter.com/slaeryan) [![](https://img.shields.io/badge/linkedin-@UpayanSaha-0084b4?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/upayan-saha-404881192/)
