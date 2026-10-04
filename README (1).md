# Local Telemetry Generation

Mini-project: Windows event logging with Sysmon and Security auditing.

**Author:** Dnyaneshwari Kale
**Programme:** B.Tech Cybersecurity, Sanjivani University
**Date:** 4 October 2026

The full write-up, with screenshots and field-by-field analysis, is in
[`Local_Telemetry_Generation_Report.pdf`](Local_Telemetry_Generation_Report.pdf).

---

## Objective

Set up local logging on a Windows machine, generate five distinct security events by hand, and show where the logs are stored. For each event, identify the three fields an analyst reads first: **timestamp**, **user** and **process**.

## Environment

| Item | Detail |
| --- | --- |
| OS | Windows, build 10.0.26100 |
| Host | Dnyaneshwari (local machine) |
| Logging utility | Sysmon v15.22, default configuration |
| Built-in auditing | Windows Security auditing via `auditpol` |
| Viewing tools | Event Viewer (Friendly and XML views), PowerShell |

---

## Process

### Step 1: Enable Windows audit policy

Run in an elevated prompt. This makes the Security log record logons and account changes.

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
```

### Step 2: Install Sysmon

Extract the Sysmon package to `C:\Sysmon`, then install it as a service and driver:

```powershell
.\Sysmon64.exe -accepteula -i
Get-Service Sysmon64
```

No configuration file was supplied, so Sysmon runs with defaults (this is why `RuleName` shows `-`).

### Step 3: Generate five events

| # | Event | How it was generated | Log | Event ID |
| --- | --- | --- | --- | --- |
| 1 | Process creation | Launch Notepad | Sysmon | 1 |
| 2 | New local user | `net user TelemetryTest <password> /add` | Security | 4720 / 4722 |
| 3 | Failed login | Log on as `TelemetryTest` with a wrong password | Security | 4625 |
| 4 | PowerShell command | `powershell.exe -NoProfile -Command "Write-Output Event4-Test"` | Sysmon | 1 |
| 5 | Command Prompt command | `cmd.exe /c "echo Event5-Test"` | Sysmon | 1 |

### Step 4: Read the logs

Open **Event Viewer** and use the Friendly or XML view. Note the timestamp, user and process fields for each event.

### Step 5: Locate the log files

| Log | File path | Event Viewer location |
| --- | --- | --- |
| Security | `C:\Windows\System32\winevt\Logs\Security.evtx` | Windows Logs > Security |
| Sysmon | `C:\Windows\System32\winevt\Logs\Microsoft-Windows-Sysmon%4Operational.evtx` | Applications and Services Logs > Microsoft > Windows > Sysmon > Operational |

### Step 6: Clean up

Remove the test accounts after the exercise:

```powershell
net user TelemetryTest /delete
net user TelemetryTest2 /delete
```

---

## Key fields per event

| Event | Timestamp field | User field | Process field |
| --- | --- | --- | --- |
| Process creation (Sysmon 1) | `UtcTime` | `User` | `Image`, `ProcessId` |
| User created (4720) | `TimeCreated` | `TargetUserName`, `SubjectUserName` | Not recorded; see `SubjectLogonId` |
| Failed login (4625) | `TimeCreated` | `TargetUserName` | `ProcessName` |
| PowerShell (Sysmon 1) | `UtcTime` | `User` | `Image`, `CommandLine` |
| cmd.exe (Sysmon 1) | `UtcTime` | `User` | `Image`, `CommandLine`, `ParentImage` |

Sysmon and Security events store time in **UTC**; Event Viewer displays **local time** (IST, UTC+5:30).

## MITRE ATT&CK mapping

| Event | Technique |
| --- | --- |
| User created | T1136.001 Create Account: Local Account |
| Failed login (when repeated) | T1110 Brute Force |
| PowerShell | T1059.001 PowerShell |
| cmd.exe | T1059.003 Windows Command Shell |

## Correlation

The logon session ID links the two logs: the 4720 event shows `SubjectLogonId 0xc7abe`, and the PowerShell and cmd.exe events show `LogonId 0xC7ABE` for the same user.

## Limitations

- Sysmon ran with its default configuration, with no custom filtering.
- One failed logon is normal; brute-force detection needs many failures in a short period.
- The test password appears in clear text in command history. This is acceptable only for a throwaway lab account.

## Repository contents

| File | Description |
| --- | --- |
| `README.md` | This file: summary of the process |
| `Local_Telemetry_Generation_Report.pdf` | Full report with screenshots and analysis |
