# Home SOC Lab: Splunk, Sysmon & Kali

---

Splunk is a software platform that collects, indexes, and analyzes machine-generated data (like logs) in real time. This platform is widely used by many enterprises and large organizations for different purposes, such as security monitoring, IT operations, compliance, business analytics, and incident response.

[![](<Splunk Home Lab/archi.png>)](<Splunk Home Lab/archi.png>)

---

In this project i will walk you through:
- **How to install splunk on docker**
- **How to install sysmon (a security camera for detailed logging and monitoring) and how to install universal forwarder (logs delivery guy) in a windows agent (victim)**
- **Stimulate attacks from the kali linux endpoint (attacker)**
- **Analyze and investigate attacks on splunk**
- **Create SPL search queries for easier search**

---

At the end of this project, you will be able to install splunk on a docker and install and configure sysmon and Universal Forwarders in your windows agent. Stimulate attacks from the kali linux endpoint and investigate these attacks in real time in splunk.

---

# What you need first (prerequisite)

- i. Windows OS running docker desktop (about 4 GB of RAM free for Docker)
- ii. Download the sysmon zip file and olafhartong.xml config file
- iii. Download a splunk universal forwarder and splunk Add-ons for Windows and Sysmon (you will need to create a splunk account if you don't have one).
- iv. Kali linux
  
  ---
**Note**: All three machines (docker host, windows, and kali) need to be able to reach each other.

---

# Step 1: Install splunk on docker
Open your terminal (cmd)
i. Pull the splunk image with this command (docker pull splunk/splunk:latest)
ii. Run a single splunk instance on docker.
docker run -d \
 - name splunk \
 -p 8000:8000 \
 -p 8089:8089 \
 -e SPLUNK_START_ARGS= - accept-license \
 -e SPLUNK_PASSWORD=Chang3d!Pass \
 splunk/splunk:latest

Splunk is big, and it takes a moment to wake up.

---

**Note**: Run docker logs -f splunk to see Docker live logs.

Finally, open your web browser and go to: http://localhost:8000

Log in with the username admin and the password (Chang3d!Pass), i.e., whatever you passed in the run command. If you see the splunk homepage, you now have a real splunk running.

--- 

# Step 2: Create an index to store your data.

In the splunk web page: **Settings → Indexes → New Index**. Name it **windows**. Click Save.

An **"index"** is just a labeled drawer. If this drawer doesn't exist, any data the forwarder sends gets thrown in the trash silently-this trips up almost every beginner, so don't skip it.

 [![](<Splunk Home Lab/Screenshot 2026-09-18 010418.png>)](<Splunk Home Lab/Screenshot 2026-09-18 010418.png>) 

 
---

# Step 3: Teach splunk how to read windows data.

Windows logs look messy until splunk knows how to parse them. You install two free "add-ons" (translation packs) on your splunk:

- Splunk add-on for microsoft windows
- Splunk add-on for sysmon

# Download both from splunkbase (splunkbase.splunk.com - free account needed).

- i. Open the command prompt and navigate to your downloads.
- ii. Copy each file into the splunk container (use the real filenames you downloaded).
- iii. Install them inside splunk in this format: docker cp <file> <container>:<destination>
- docker cp splunk-add-on-for-sysmon_501.spl splunk:/tmp/
- docker cp splunk-add-on-for-microsoft-windows_1102.spl splunk:/tmp/

[![](<Splunk Home Lab/Screenshot 2026-10-06 024021.png>)](<Splunk Home Lab/Screenshot 2026-10-06 024021.png>) 

[![](<Splunk Home Lab/Screenshot 2026-09-16 001748.png>)](<Splunk Home Lab/Screenshot 2026-09-16 001748.png>)

---

# Step 4: Install the Sysmon zip file and olafhartong.xml config file.

Sysmon (a free microsoft sysinternal) is a deep windows endpoint telemetry to the windows event log, tuned via xml config, and the logs are sent to splunk for detection and forensics. It needs a good config to be useful.

- i. Download sysmon from microsoft sysinternals (https://download.sysinternals.com/files/Sysmon.zip).

- ii. Download the config file sysmonconfig-export.xml from (https://github.com/olafhartong/sysmon-modular).

- iii. Put both in the same folder, open the command prompt as admin, go to that folder, and run: **sysmon64.exe -accepteula -i sysmonconfig-export.xml**. That installs sysmon as a background service. It's now recording. You can confirm it's alive with: sc query sysmon64

[![](<Splunk Home Lab/Screenshot 2026-09-15 205007.png>)](<Splunk Home Lab/Screenshot 2026-09-15 205007.png>)

--- 

# Step 5: Turn on extra Windows logging.

Windows hides some of its best evidence by default. Turn it on so you capture the commands attackers type, not just that "something ran."

Open **gpedit.msc** (Local Group Policy Editor) and enable these:

- **Computer Configuration** → Windows Settings → Security Settings → Local Policies → Audit Policy → Audit Process Creation → set to Success.

- **Computer Configuration** → Windows Settings → Security Settings → Advanced Audit Policy → Audit Logon events → set to Success and Failure. (This is what makes event 4624 and event 4625 security logs show.)

- **Computer Configuration** → Administrative Templates → System → Audit Process Creation → Include command line in process creation events → Enabled. (This is what makes event 4688 actually show the full command.)

- **Computer Configuration** → Administrative Templates → Windows Components → Windows PowerShell → Turn on PowerShell Script Block Logging → Enabled.

Then run gpupdate /force in an admin Command Prompt to apply it.

[![](<Splunk Home Lab/policy.png>)](<Splunk Home Lab/policy.png>) missing

---

# Step 6: Install and set up the Universal Forwarder.

Universal Forwarder (UF) is a lightweight agent that sends logs/data from a host to a Splunk indexer-no web UI, no indexing, just forwarding.

Install location: **C:\Program Files\SplunkUniversalForwarder**\

Config folder (all config files live here): …**\etc\system\local**\

**A. Install (GUI method)**

- **i. Download splunkforwarder-9.x.x-x64-release.msi from splunk.com**.
- **ii. Run the MSI and accept the license**.
- **iii. Set admin username/password for the UF**.
- **iv. Deployment Server-leave the box unchecked**
- **v. Receiving Indexer-enter <Docker host_ip>:9997**
- **vi. Finish-service starts automatically**.

[![](<Splunk Home Lab/Screenshot 2026-09-15 222911.png>)](<Splunk Home Lab/Screenshot 2026-09-15 222911.png>) 

B. **Set forwarder (separate from splunk's login) via user-seed.conf in \etc\system\local**\:

[user_info]

**USERNAME** = **admin**

**PASSWORD** = **NxxPass32xxx!** (remember to add yours)

(Stop forwarder → delete etc\passwd → start, so the seed is read.)

**C. outputs.conf**-tell forwarder where to send data:

[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.xxx.x.xxx:9997 (host IP:destination port)


D. **inputs.conf** - tell forwarder what to collect (security, system, application, Sysmon, PowerShell), all with index = windows (remember it is where splunk stores all data that comes from the host):

[![](<Splunk Home Lab/outin.png>)](<Splunk Home Lab/outin.png>)


E. **Grant Sysmon channel read access (the forwarder runs as a limited account NT Service\SplunkForwarder)**:

Open your terminal and run:

**wevtutil sl Microsoft-Windows-Sysmon/Operational /ca:"O:BAG:SYD:(A;;0xf0007;;;SY)(A;;0x7;;;BA)(A;;0x1;;;BO)(A;;0x1;;;SO)(A;;0x1;;;LS)(A;;0x1;;;S-1–5–32–573)"
net localgroup "Event Log Readers" "NT SERVICE\SplunkForwarder" /add**


F. **Restart the universal forwarder**

"C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" restart

H. **Verify if it is working**

"C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" list forward-server -auth admin:Nexx321xx!(password)

Check if it is working on splunk - index=windows | stats count by sourcetype source OR index=windows source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"

**Note**:

- **i. All forwarder config files must be in etc/system/local -anywhere else is ignored**.

- **ii. The forwarder is its own program: own password, own service account (NT SERVICE\SplunkForwarder)**.

- **iii. The forwarder's limited service account needs explicit permission to read locked-down channels like sysmon to avoid errorCode=5, i.e., splunk been unable to capture sysmon event logs**.

---

# The Attacks-what they are and how they were caught

All attacks below are non-destructive-they either create nothing or create one harmless file that gets deleted afterward. Each one teaches a different kind of detection.

# Attack 1- System discovery (systeminfo)

**What it is**: dumps the operating system version, patch level, and hardware details. Attackers run this first to figure out which exploits might work.

**Run**: systeminfo, whoami / priv

**Caught by**: index=windows source="*Sysmon*" EventCode=1 Image="*systeminfo OR whoami / priv*"

**Event type**: Sysmon EventCode 1-"a new program started." Shows the full command, who ran it, and its parent process.

**Mitre att&ck**: T1082 & T1033

[![systeminfo](<Splunk Home Lab/systeminfo.png>)](<Splunk Home Lab/systeminfo.png>)

[![whoami/priv](<Splunk Home Lab/whoami.png>)](<Splunk Home Lab/whoami.png>)

---

# Attack 2- Account Discovery (net user, net local group administrators) 

**What it is**: lists who has an account on the machine and who's an administrator. Attackers use this to find accounts worth stealing.

**Run**: net user, net local group administrators

**Caught by**: index=windows source="*Sysmon*" EventCode=1 (CommandLine="*net user*" OR CommandLine="*localgroup*")

**Mitre att&ck**: T1087

[![net user cmd](<Splunk Home Lab/netcmd.png>)](<Splunk Home Lab/netcmd.png>)

[![net user splunk](<Splunk Home Lab/netspl.png>)](<Splunk Home Lab/netspl.png>)

---

# Attack 3 - Network Discovery (ipconfig /all, arp -a, netstat -ano)

**What it is**: maps out the network-IP address, recently contacted devices, and active connections. Used to find other machines to move to.

**Caught by**: index=windows source="*Sysmon*" EventCode=1 (CommandLine="*ipconfig*" OR CommandLine="*arp*" OR CommandLine="*netstat*")

**Mitre att&ck**: T1016 & T1049

[![](<Splunk Home Lab/spipall.png>)](<Splunk Home Lab/spipall.png>)

[![](<screenshots/Screenshot 2026-06-01 202704.png>)](<screenshots/Screenshot 2026-06-01 202704.png>) missing

---

# Attack 4 - Disguised PowerShell

**What it is**: 
Running PowerShell the way real malware does - Base64-encoded so a human can't read the command at a glance, with flags (-nop -w hidden -ep bypass) that hide the window and skip safety checks.

**A.Encoded command (-enc)**

Run this command on your terminal:

**powershell -enc VwByAGkAdABlAC0ASABvAHMAdAAgACIAaABlAGwAbABvACIA**

- **Technique**: Base64-encoded PowerShell command

- **Decoded payload**: "Write-Host "hello"

- **Purpose**: Obfuscate the command to evade simple string-based detection

- **MITRE ATT&CK**:

T1059.001 - Command & Scripting Interpreter: PowerShell

T1027 - Obfuscated Files or Information

**Caught by**: index=windows source="*Sysmon*" EventCode=1 (CommandLine="*-enc*" OR CommandLine="*bypass*" OR CommandLine="*hidden*")

[![](<Splunk Home Lab/enccmd.png>)](<Splunk Home Lab/enccmd.png>)

[![](<Splunk Home Lab/encspl.png>)](<Splunk Home Lab/encspl.png>) 

[![script logging](<Splunk Home Lab/hello world scr.png>)](<Splunk Home Lab/hello world scr.png>) 

**B. Steathly execution**

Run this on your command terminal:

**powershell -nop -w hidden -ep bypass -c "whoami"**

- **Technique**: Hidden window + execution policy bypass + no profile

- **Purpose**: Silently run recon without user visibility or policy restrictions

- **MITRE ATT&CK**:
T1059.001 - PowerShell

T1033 - System Owner/User Discovery (whoami)

T1562.001 - Impair Defenses (bypass execution policy)

T1564.003 - Hide Artifacts (hidden window)

**Caught by**: index=windows EventCode=4104

Even though the attacker scrambled the command to hide it, windows records the actual code that ran. Inside that event's ScriptBlockText field, the real command - Write - Host "hello"- was visible, fully decoded. This proves that encoding a command does not defeat proper logging.

[![](<Splunk Home Lab/encspl.png>)](<Splunk Home Lab/encspl.png>)

---

# Attack 5 - LOLBin download attempt

**What it is**:
windows defender blocked the download and flagged it as a threat (confirmed with **Get-MpThreatDetection**). This was a bonus result - instead of just seeing the attack, the lab also captured the defense catching it in real time. That's exactly the "**attack → block → log**" chain a real SOC investigates.

[![](<Splunk Home Lab/Get MP.png>)](<Splunk Home Lab/Get MP.png>) 

---

# Attack 6 - Port Scan (from Kali)

**What it is**: 
knocking on every "door" (port) of the windows machine to see what's open and running - usually the first thing an outside attacker does.
Note: This one produced zero results in Splunk. Not a bug - the Olaf Hartong Sysmon configuration deliberately filters out most network-connection logging (EventCode 3) to reduce noise, so ordinary scan traffic often isn't logged by design. **A good lesson**: not every attack will show up depending on how your detection tool is configured - that's a real, common finding in actual security work.

**nmap -sV 192.xxx.0.xxx**

[![nmap scan](<Splunk Home Lab/Screenshot 2026-10-01 231401.png>)](<Splunk Home Lab/Screenshot 2026-10-01 231401.png>) 

---

# Attack 7 - SMB Brute-Force Login (the complete story)

**What it is**:

Trying to log into the windows machine over the network (SMB) using a list of guessed passwords - like trying different keys in a lock.

[![smb brute force](<Splunk Home Lab/nxc smb.png>)](<Splunk Home Lab/nxc smb.png>) 

**Failed attempts**:
nxc smb 192.xxx.0.xxx -u testuser -p wrong1 wrong2 wrong3

**Successful login (using the correct password)**:
nxc smb 192.xxx.0.xxx -u testuser -p correct password

**Caught by**:

index=windows (EventCode=4624 OR EventCode=4625) Account_Name="testuser" | table _time EventCode src_ip | sort _time

**The result - a textbook brute-force pattern**:

10:30:16 4625 FAILURE (wrong1)

10:30:16 4625 FAILURE (wrong2)

10:30:16 4625 FAILURE (wrong3)

10:30:37 4624 SUCCESS ← the break-in

Three failures followed by a success, all from the same attacker IP within seconds - this is exactly the pattern that would page a real security analyst in the middle of the night. It was also confirmed that a remote SMB login shows **Logon_Type = 3**, which distinguishes a network login from someone physically at the keyboard (**Logon_Type = 2**).

**Note**: A throwaway account (**testuser**) was created specifically for this test and deleted afterward with net user testuser /delete, so nothing risky touched the real windows accounts.

 ---
 
# Troubleshooting Log - Every Problem and How It Was Fixed

[![](<Splunk Home Lab/Screenshot 2026-10-02 013526.png>)](<Splunk Home Lab/Screenshot 2026-10-02 013526.png>) 

[![](<Splunk Home Lab/Screenshot 2026-10-02 013807.png>)](<Splunk Home Lab/Screenshot 2026-10-02 013807.png>) 

---

# Important Lessons Learned

1. **Config files only work in the exact right folder**. For the Universal Forwarder, that folder is always etc\system\local\. A file saved anywhere else is silently ignored - no error, it just does nothing. This single mistake caused several of the problems above.

2. **The Universal Forwarder is its own separate program**. It has its own login (different from the Splunk web password) and its own Windows service account. Don't assume they're the same thing.

3. **Some Windows log channels are locked down tighter than others**. Security, System, and Application logs are easy to read. Sysmon and PowerShell Operational logs need extra permission granted explicitly, or the forwarder gets silently blocked with an "Access Denied" (errorCode=5).

4. **If you change a password, change it everywhere it's used**. Changing Splunk's admin password without also updating it inside docker-compose.yml caused the whole container to fail its startup routine on the next reboot.

5. **Docker Compose ties your data to the folder name**. Renaming a project folder can quietly create brand-new, empty storage instead of reusing the old one - even though nothing was technically deleted. Renaming a Compose project folder is risky and should be avoided once real data is inside it.

6. **Encoding a command does not hide it from good logging**. Attackers scramble PowerShell commands with Base64 to avoid detection. Basic process logging (EventCode 1) only sees the scrambled version - but PowerShell's own Script Block Logging (EventCode 4104) records what the command actually does, defeating the disguise completely.

7. **A missing audit policy means a missing detection - with no error message**. Windows will happily reject a bad login (and the attacker will see it fail) without writing any log at all, if the "Audit Logon" policy isn't turned on. This is invisible until you specifically check for it - there's no obvious error pointing you to the cause.

8. **Not every attack will be visible, and that's normal**. The port scan (Attack 6) produced no logs because of how the Sysmon configuration is tuned to reduce noise. Real security tools make trade-offs between seeing everything and drowning in irrelevant data - understanding why something isn't logged is as valuable as understanding what is.

9. **When something that worked yesterday breaks today, check what changed - especially after a reboot**. Several of the hardest problems in this lab (the password mismatch, the disappearing index, the restart loop) only appeared after a reboot, because reboots force things to restart fresh and expose configuration that had drifted out of sync.

10. **Reading the actual error message beats guessing**. Every real breakthrough in this lab came from reading a specific log line - errorCode=5, 401 Unauthorized, Access is denied, "no matching events found" - rather than assuming a cause. This is the core habit of real troubleshooting and real security analysis.
