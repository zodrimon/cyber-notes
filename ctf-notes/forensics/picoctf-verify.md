## PicoCTF - Verify (Forensics) Writeup
**Author:** roninxyg

### 1. Connecting to the Server
First, I connected via SSH to the challenge server:

```bash
ssh -p 58601 ctf-player@rhea.picoctf.net

```

I typed `yes` to accept the fingerprint, entered the password `1db87a14`, and logged in.

### 2. Recon and Exploring the Directories

Once inside, I checked the contents of the home directory:

```bash
ctf-player@pico-chall$ ls
checksum.txt  decrypt.sh  files

```

I moved into the `files` folder to see what I was dealing with:

```bash
ctf-player@pico-chall$ cd files
ctf-player@pico-chall$ ls

```

The screen flooded with hundreds of randomly named files. There was no way I was checking these one by one, so I backed up to the main directory to set up an exploit:

```bash
ctf-player@pico-chall$ cd ..

```

### 3. Finding the Flag

Back in the main folder where `decrypt.sh` lives, I ran a `for` loop to automatically feed every file into the decryption script and filter for the flag:

```bash
ctf-player@pico-chall$ for i in $(find files); do ./decrypt.sh $i | grep pico; done

```

The terminal spammed `bad magic number` for the fakes until it hit the correct file and printed the flag:

```text
bad magic number
bad magic number
picoCTF{trust_but_verify_2cdcb2de}

```

**Flag Found:** `picoCTF{trust_but_verify_2cdcb2de}`

---

## How to Actually Learn to Script This Yourself

To write these on the fly without looking up guides, break the problem down into three steps:

### 1. Test One Target

Never write a loop without testing the command manually on a single file first.

```bash
./decrypt.sh files/0SgkM1fC

```

This tells you the exact format the script expects (`files/filename`) and what a failure looks like.

### 2. Use the One-Line Loop Template

Memorize this exact syntax structure. It handles 90% of basic CTF automation tasks:

```bash
for x in LIST; do COMMAND $x; done

```

* **`x`**: A temporary variable for the current file.
* **`LIST`**: The target files, generated using `$(find files)` or `files/*`.
* **`COMMAND $x`**: The action you want to take, using your variable (`./decrypt.sh $i`).

### 3. Pipe and Filter

Since running the raw loop spams the terminal with noise, tack `| grep pico` onto the command inside the loop. This hides the errors and only outputs the line containing the flag format.

If you force yourself to build scripts piece-by-piece using this layout, it will become muscle memory.

```

```
