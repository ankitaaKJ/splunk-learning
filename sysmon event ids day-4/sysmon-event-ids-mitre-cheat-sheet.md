# Sysmon Event IDs & MITRE ATT&CK Detection Cheat Sheet

**Log Path:** `Applications and Services Logs/Microsoft/Windows/Sysmon/Operational`  
**Timestamps:** Always recorded in **UTC**  
**Schema Reference:** Sysmon v15.22 (`schemaversion="4.82"`)

---

## 1. Master Sysmon Event IDs Reference (IDs 1–29 & 255)

### Process & Thread Activity
| ID | Event Name | XML Filter Tag | Description & Key Telemetry |
| :--- | :--- | :--- | :--- |
| **1** | Process creation | `ProcessCreate` | Logs new processes with full command line, parent process, `ProcessGuid`, `SessionGuid`, `IntegrityLevel`, and file hashes. |
| **5** | Process terminated | `ProcessTerminate` | Reports when a process ends (`UtcTime`, `ProcessGuid`, `ProcessId`, `Image`). |
| **8** | CreateRemoteThread | `CreateRemoteThread` | Detects a process creating a thread in another process (code injection). Captures `SourceImage`, `TargetImage`, `StartAddress`, `StartModule`, and `StartFunction`. |
| **10** | ProcessAccess | `ProcessAccess` | Logs when a process opens another process (e.g., credential dumping via `lsass.exe`). Captures `GrantedAccess` and `CallTrace`. |
| **25** | ProcessTampering | `ProcessTampering` | Detects process hiding techniques such as process hollowing or herpaderping (`Type`: `Image is replaced` / `Image is locked`). |

### File System & Storage Activity
| ID | Event Name | XML Filter Tag | Description & Key Telemetry |
| :--- | :--- | :--- | :--- |
| **2** | File creation time changed | `FileCreateTime` | Detects explicit modification of a file's creation timestamp (timestomping). Logs `CreationUtcTime` vs. `PreviousCreationUtcTime`. |
| **9** | RawAccessRead | `RawAccessRead` | Detects raw read operations from a drive using the `\\.\` denotation (bypassing file locks for `NTDS.dit`/`SAM`). |
| **11** | FileCreate | `FileCreate` | Logs when a file is created or overwritten (crucial for Startup, Temp, Downloads, and web root directories). |
| **15** | FileCreateStreamHash | `FileCreateStreamHash` | Logs named file streams (e.g., `Zone.Identifier` Mark of the Web on downloads) along with the file hash and stream contents. |
| **23** | FileDelete | `FileDelete` | Logs file deletion and archives a copy of the deleted file into `ArchiveDirectory` (`C:\Sysmon` by default). |
| **26** | FileDeleteDetected | `FileDeleteDetected` | Logs file deletion without archiving the deleted file to disk. |
| **27** | FileBlockExecutable | `FileBlockExecutable` | Logs when Sysmon detects and blocks the creation of executable (PE) files. |
| **28** | FileBlockShredding | `FileBlockShredding` | Logs when Sysmon detects and blocks file shredding tools (e.g., SDelete). |
| **29** | FileExecutableDetected | `FileExecutableDetected` | Logs when Sysmon detects the creation of a new executable (PE) file. |

### Network & DNS Activity
| ID | Event Name | XML Filter Tag | Description & Key Telemetry |
| :--- | :--- | :--- | :--- |
| **3** | Network connection | `NetworkConnect` | Logs TCP/UDP connections linked to a process via `ProcessGuid` (disabled by default; requires explicit config rules). |
| **22** | DNSEvent (DNS query) | `DnsQuery` | Logs when a process executes a DNS query, regardless of success, failure, or caching (`QueryName`, `QueryResults`). |

### Modules, Drivers & Registry
| ID | Event Name | XML Filter Tag | Description & Key Telemetry |
| :--- | :--- | :--- | :--- |
| **6** | Driver loaded | `DriverLoad` | Logs kernel driver loads along with file hashes (`Hashes`) and asynchronous signature status (`Signed`, `Signature`, `SignatureStatus`). |
| **7** | Image loaded | `ImageLoad` | Logs DLL/module loads inside a process (disabled by default; high volume — filter carefully). |
| **12** | RegistryEvent (Create/Delete) | `RegistryEvent` | Logs registry key and value creation or deletion (`HKLM`, `HKU`, `HKCR`, `HKLM\System\CurrentControlSet`). |
| **13** | RegistryEvent (Value Set) | `RegistryEvent` | Logs registry value modifications, recording the written `DWORD`, `QWORD`, or string data in `Details`. |
| **14** | RegistryEvent (Rename) | `RegistryEvent` | Logs registry key and value rename operations, recording the new name in `NewName`. |

### WMI, Named Pipes & Clipboard
| ID | Event Name | XML Filter Tag | Description & Key Telemetry |
| :--- | :--- | :--- | :--- |
| **17** | PipeEvent (Pipe Created) | `PipeEvent` | Logs creation of a named pipe (`PipeName`), commonly used for interprocess C2 communication. |
| **18** | PipeEvent (Pipe Connected) | `PipeEvent` | Logs when a client process connects to a named pipe server. |
| **19** | WmiEvent (Filter) | `WmiEvent` | Logs registration of a WMI event filter (`EventNamespace`, `Name`, and WQL `Query` expression). |
| **20** | WmiEvent (Consumer) | `WmiEvent` | Logs registration of a WMI event consumer (`Name`, `Type`, `Destination`). |
| **21** | WmiEvent (ConsumerToFilter) | `WmiEvent` | Logs when a WMI consumer binds to a WMI filter (`Consumer`, `Filter`). |
| **24** | ClipboardChange | `ClipboardChange` | Logs when the system clipboard contents change (`Session`, `ClientInfo`, `Hashes`). |

### Sysmon Service & System Events
| ID | Event Name | XML Filter Tag | Description & Key Telemetry |
| :--- | :--- | :--- | :--- |
| **4** | Sysmon service state changed | *N/A (Cannot be filtered)* | Reports when the Sysmon service starts or stops (`State`). |
| **16** | ServiceConfigurationChange | *N/A (Cannot be filtered)* | Logs updates to the Sysmon XML configuration or filtering rules (`Configuration`, `ConfigurationFileHash`). |
| **255** | Error | *N/A* | Logs internal Sysmon errors, heavy load drops, or integrity/security issues. |

---

## 2. High-Value MITRE ATT&CK Detection Mapping

| Sysmon ID & Name | MITRE ATT&CK Technique | Detection Use Case & Key Fields |
| :--- | :--- | :--- |
| **1: Process Creation** | **T1059** (Command & Scripting Interpreter)<br>**T1218** (System Binary Proxy Execution) | **Use Case:** Catching encoded PowerShell, LOLBins (`mshta.exe`, `rundll32.exe`, `certutil.exe`), or Office apps spawning shells.<br>**Look at:** `ParentImage`, `CommandLine`, `Hashes`, `IntegrityLevel` |
| **7: Image Loaded** | **T1574.001 / .002** (DLL Search Order Hijacking / Side-Loading)<br>**T1055.001** (DLL Injection) | **Use Case:** Unsigned or weirdly signed DLLs loaded from writable user paths (`AppData`, `Temp`, `Public`) by trusted binaries.<br>**Look at:** `Image`, `ImageLoaded`, `Signed`, `SignatureStatus` |
| **8: CreateRemoteThread** | **T1055** (Process Injection) | **Use Case:** An untrusted or user-land process injecting a thread into a sensitive system process (`explorer.exe`, `svchost.exe`).<br>**Look at:** `SourceImage`, `TargetImage`, `StartFunction` |
| **25: ProcessTampering** | **T1055.012** (Process Hollowing)<br>**T1055.013** (Process Doppelgänging) | **Use Case:** Detecting when the mapped image in memory doesn't match the file on disk (hollowing/herpaderping).<br>**Look at:** `Image`, `Type` (`Image is replaced` / `Image is locked`) |
| **10: ProcessAccess** | **T1003.001** (OS Credential Dumping: LSASS Memory) | **Use Case:** Non-system processes opening `lsass.exe` with sensitive access masks (`0x1010`, `0x1410`, `0x1438`, `0x1FFFFF`).<br>**Look at:** `SourceImage`, `TargetImage` (`lsass.exe`), `GrantedAccess`, `CallTrace` |
| **9: RawAccessRead** | **T1003.003** (NTDS)<br>**T1006** (Direct Volume Access) | **Use Case:** User-mode processes reading `\\.\` raw disk volumes directly to bypass file locks and copy `NTDS.dit` or `SAM`.<br>**Look at:** `Image`, `Device` |
| **12, 13, 14: RegistryEvent** | **T1547.001** (Registry Run Keys / Startup Folder)<br>**T1546.015** (COM Hijacking)<br>**T1562.001** (Impair Defenses) | **Use Case:** Modifications to `Run`/`RunOnce` keys, `Image File Execution Options` (IFEO) debugger hijacks, or Defender tamper keys.<br>**Look at:** `EventType`, `TargetObject`, `Details` |
| **19, 20, 21: WmiEvent** | **T1546.003** (WMI Event Subscription) | **Use Case:** Fileless persistence via permanent WMI Event Filters bound to CommandLine or ActiveScript consumers.<br>**Look at:** `Operation`, `Name`, `Query` (ID 19), `Destination` (ID 20) |
| **6: Driver Loaded** | **T1068** (Exploitation for Privilege Escalation)<br>**T1014** (Rootkit) | **Use Case:** Bring Your Own Vulnerable Driver (BYOVD) attacks loading known vulnerable signed drivers from `Temp` or `Downloads`.<br>**Look at:** `ImageLoaded`, `Hashes`, `Signed`, `Signature` |
| **11: FileCreate** | **T1105** (Ingress Tool Transfer)<br>**T1505.003** (Web Shell) | **Use Case:** Dropping executables/scripts into `Startup` folders, `C:\Users\Public`, or web root directories (`inetpub\wwwroot`).<br>**Look at:** `Image`, `TargetFilename` |
| **15: FileCreateStreamHash** | **T1553.005** (Mark-of-the-Web Bypass)<br>**T1564.004** (NTFS File Attributes) | **Use Case:** Tracking downloaded payloads via `Zone.Identifier` ADS (`ZoneId=3` = Internet) or hiding payloads in Alternate Data Streams.<br>**Look at:** `TargetFilename` (`:Zone.Identifier`), `Contents`, `Hash` |
| **2: FileCreateTime** | **T1070.006** (Timestomp) | **Use Case:** Adversaries altering file creation timestamps to blend dropped malware into `System32` alongside legitimate OS files.<br>**Look at:** `Image`, `TargetFilename`, `CreationUtcTime`, `PreviousCreationUtcTime` |
| **23 / 26: FileDelete** | **T1070.004** (Indicator Removal: File Deletion) | **Use Case:** Post-exploitation cleanup of dropped tools or staging scripts (ID 23 also preserves a forensic copy in `C:\Sysmon`).<br>**Look at:** `Image`, `TargetFilename`, `Hashes`, `Archived` |
| **3: NetworkConnect** | **T1071** (Application Layer Protocol)<br>**T1021** (Remote Services) | **Use Case:** Living-off-the-land binaries (`rundll32.exe`, `regsvr32.exe`, `notepad.exe`) making outbound C2 or internal SMB/RDP connections.<br>**Look at:** `Image`, `Initiated`, `DestinationIp`, `DestinationPort` |
| **22: DnsQuery** | **T1071.004** (DNS)<br>**T1568.002** (Domain Generation Algorithms) | **Use Case:** High-entropy DGA domains, DNS tunneling, or non-browser processes resolving suspicious TLDs.<br>**Look at:** `Image`, `QueryName`, `QueryResults` |
| **17, 18: PipeEvent** | **T1055** (Process Injection / Beacon IPC)<br>**T1021.002** (SMB/Windows Admin Shares) | **Use Case:** Default or malleable C2 named pipe patterns (e.g., Cobalt Strike `\msagent_`, `\postex_`, or PsExec `\PSEXESVC`).<br>**Look at:** `EventType`, `PipeName`, `Image` |

---

## 3. High-Signal Correlation Chains & Quick Reference

### Multi-Event Correlation Chains (Pivot via `ProcessGuid`)
1. **Phishing Payload Execution:** `ID 15` (`:Zone.Identifier` created on archive/ISO) $\rightarrow$ `ID 1` (Office/Browser spawns `cmd.exe` or `powershell.exe`) $\rightarrow$ `ID 3` (Outbound C2 connection).
2. **LSASS Credential Dump:** `ID 10` (`GrantedAccess=0x1010` on `lsass.exe` from unsigned binary) $\rightarrow$ `ID 11` (`.dmp` file created in `Temp`) $\rightarrow$ `ID 23` (`.dmp` deleted after exfiltration).
3. **Cobalt Strike Beacon:** `ID 8` (`CreateRemoteThread` into `rundll32.exe`) $\rightarrow$ `ID 17` (Named pipe created by `rundll32.exe`) $\rightarrow$ `ID 3` (Periodic outbound HTTPS beaconing).

### CLI & Configuration Cheat Sheet
* **Install with config:** `sysmon64.exe -accepteula -i config.xml`
* **Update live config:** `sysmon64.exe -c config.xml`
* **Dump active config:** `sysmon64.exe -c`
* **Print XML schema:** `sysmon64.exe -s`
* **Registry Root Abbreviations:** `HKLM` (`HKEY_LOCAL_MACHINE`), `HKU` (`HKEY_USERS`), `HKCR` (`HKEY_LOCAL_MACHINE\Classes`), `HKLM\System\CurrentControlSet` (`ControlSet00x`).
