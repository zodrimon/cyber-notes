# Windows Privilege Escalation Notes[Draft]

> These are my personal cybersecurity learning notes written in my own words.
> I do not publish flags, direct answers, private-room solutions, or copied premium content.

## Platform
TryHackMe

## Topic
Windows Privilege Escalation

## Goal
Understand how privilege escalation works on Windows systems by learning enumeration, permissions, misconfigurations, services, users, groups, and privilege abuse.

## What I learned today
- Windows privilege escalation starts with proper enumeration.
- I should understand my current user before trying to escalate.
- Services, permissions, users, groups, scheduled tasks, and stored credentials are important areas to check.
- Manual enumeration helps me understand what tools are doing behind the scenes.

## Basic enumeration commands

```cmd
whoami
whoami /priv
whoami /groups
hostname
systeminfo
ver
ipconfig /all
net user
net localgroup
net localgroup administrators
netstat -ano
sc query
```

## Important areas to check
# User context

Before privilege escalation, I need to know which user I am and what privileges/groups I have.

## System information

OS version, architecture, patches, and hostname can help identify possible attack paths.

## Services

Misconfigured services can sometimes allow privilege escalation.

# File permissions

Weak permissions on important files or folders may allow modification or abuse.

# Stored credentials

Credentials may be stored in files, registry, config files, history, or scripts.

# My mistakes
I should not jump directly to tools.
I should understand each command before using it.
I should take clean notes while solving the room.
I should separate public notes from private room answers.
# summary

Windows privilege escalation is mainly about enumeration and finding weak points in permissions, services, credentials, and user privileges. The goal is to move from a low-privileged user to a higher-privileged user by understanding how the system is configured.

# Tags
#windows #privilege-escalation #tryhackme #crta-prep #redteam
