<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/hibahmad30/threat-hunting-scenario-tor/blob/main/threat-hunting-scenario-tor-event-creation.md)

## Platforms and Languages Leveraged
- Windows 10 Virtual Machines (Microsoft Azure)
- EDR Platform: Microsoft Defender for Endpoint
- Kusto Query Language (KQL)
- Tor Browser

##  Scenario

Management suspects that some employees may be using TOR browsers to bypass network security controls because recent network logs show unusual encrypted traffic patterns and connections to known TOR entry nodes. Additionally, there have been anonymous reports of employees discussing ways to access restricted sites during work hours. The goal is to detect any TOR usage and analyze related security incidents to mitigate potential risks. If any use of TOR is found, notify management.

### High-Level TOR-Related IoC Discovery Plan

- **Check `DeviceFileEvents`** for any `tor(.exe)` or `firefox(.exe)` file events.
- **Check `DeviceProcessEvents`** for any signs of installation or usage.
- **Check `DeviceNetworkEvents`** for any signs of outgoing connections over known TOR ports.

---

## Steps Taken

### 1. Searched the `DeviceFileEvents` Table

Searched for any file that had the string "tor" in it and discovered what looks like the user "hiba-cyber" downloaded a TOR installer, did something that resulted in many TOR-related files being copied to the desktop, and the creation of a file called `tor-shopping-list.txt` on the desktop at `2025-03-07T02:39:56.0958523Z`. These events began at `2025-03-07T02:19:09.5668924Z`.

**Query used to locate events:**

```kql
DeviceFileEvents  
| where DeviceName == "hiba-windows-la"  
| where InitiatingProcessAccountName == "hiba-cyber"  
| where FileName contains "tor"  
| where Timestamp >= datetime(2025-03-07T02:19:09.5668924Z)  
| order by Timestamp desc  
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, SHA256, Account = InitiatingProcessAccountName
```
<img width="1212" alt="image" src="https://github.com/user-attachments/assets/2f600c12-a604-40aa-8da3-10bdde03af41">

---

### 2. Searched the `DeviceProcessEvents` Table

Searched for any `ProcessCommandLine` that contained the string "tor-browser-windows-x86_64-portable-14.0.7.exe". Based on the logs returned, at `2025-03-07T02:21:21.3381328Z`, an employee on the "hiba-windows-la" device ran the file `tor-browser-windows-x86_64-portable-14.0.7.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

```kql

DeviceProcessEvents  
| where DeviceName == "hiba-windows-la"  
| where ProcessCommandLine contains "tor-browser-windows-x86_64-portable-14.0.7.exe"  
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine
```
<img width="1212" height="135" alt="image" src="https://github.com/user-attachments/assets/4cd3e6dc-35fc-49a9-86b3-dd2b54e782ee">


---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched for any indication that user "hiba-cyber" actually opened the TOR browser. There was evidence that they did open it at `2025-03-07T02:25:07.9946559Z`. There were several other instances of `firefox.exe` (TOR) as well as `tor.exe` spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents  
| where DeviceName == "hiba-windows-la"  
| where FileName has_any ("tor.exe", "firefox.exe", "tor-browser.exe")  
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine  
| order by Timestamp desc
```
<img width="1212" alt="image" src="https://github.com/user-attachments/assets/2610473b-de9f-4fe6-acfe-8ff6301e1d59">



---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `2025-03-07T02:25:25.8847728Z`, an employee on the "hiba-windows-la" device successfully established a connection to the remote IP address `157.90.112.145` on port `9001`. The connection was initiated by the process `tor.exe`, located in the folder `c:\users\hiba-cyber\desktop\tor browser\browser\torbrowser\tor\tor.exe`. There were a couple of other connections to sites over port `443`.

**Query used to locate events:**

```kql
DeviceNetworkEvents  
| where DeviceName == "hiba-windows-la"  
| where InitiatingProcessAccountName != "system"  
| where InitiatingProcessFileName in ("tor.exe", "firefox.exe")  
| where RemotePort in ("9001", "9030", "9040", "9050", "9051", "9150", "80", "443")  
| project Timestamp, DeviceName, InitiatingProcessAccountName, ActionType, RemoteIP, RemotePort, RemoteUrl, InitiatingProcessFileName, InitiatingProcessFolderPath  
| order by Timestamp desc
```
<img width="1212" alt="image" src="https://github.com/user-attachments/assets/bd36fa47-02b6-434e-a4ce-cff81c5af4fa">


---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2025-03-07T02:19:09.5668924Z`
- **Event:** The user "hiba-cyber" downloaded a file named `tor-browser-windows-x86_64-portable-14.0.7.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\hiba-cyber\Downloads\tor-browser-windows-x86_64-portable-14.0.7.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2025-03-07T02:21:21.3381328Z`
- **Event:** The user "hiba-cyber" executed the file `tor-browser-windows-x86_64-portable-14.0.7.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.0.7.exe /S`
- **File Path:** `C:\Users\hiba-cyber\Downloads\tor-browser-windows-x86_64-portable-14.0.7.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2025-03-07T02:25:07.9946559Z`
- **Event:** User "hiba-cyber" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\hiba-cyber\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `2025-03-07T02:25:25.8847728Z`
- **Event:** A network connection to IP `157.90.112.145` on port `9001` by user "hiba-cyber" was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `c:\users\hiba-cyber\desktop\tor browser\browser\torbrowser\tor\tor.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `2025-03-07T02:25:25.9775303Z` - Connected to `62.112.9.92` on port `443`.
  - `2025-03-07T02:25:34.610541Z` - Local connection to `127.0.0.1` on port `9150`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "hiba-cyber" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2025-03-07T02:39:56.0958523Z`
- **Event:** The user "hiba-cyber" created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\hiba-cyber\Desktop\tor-shopping-list.txt`

---

## Summary

The user "hiba-cyber" on the "hiba-windows-la" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `hiba-windows-la` by the user `hiba-cyber`. The device was isolated, and the user's direct manager was notified.

---
