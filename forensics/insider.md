# Insider

October 2, 2026

I am doing [Insider on CyberDefenders](https://cyberdefenders.org/blueteam-ctf-challenges/insider/). Using Kali on WSL2.

Finished all 11 questions. Scrappy notes for now, organize later. These are based on what I inspected in FTK and the outputs I recorded, not a fresh re-examination of every artifact.

## Getting the evidence ready

First I extracted the zip in my Downloads folder:

```text
unzip 64-Insider.zip
Archive:  64-Insider.zip
[64-Insider.zip] temp_extract_dir/c46-FirstHack/FirstHack.ad1 password:
  inflating: temp_extract_dir/c46-FirstHack/FirstHack.ad1
  inflating: temp_extract_dir/c46-FirstHack/FirstHack.ad1.txt
```

Password: `cyberdefenders.org`. Nothing shows when typing it, thats normal.

okay, we have the files now.

Next I hashed the zip:

```text
sha256sum 64-Insider.zip
a4df81f5fce144003df36b239ff066c3282807bb2253017d354d7f6bab4a9212  64-Insider.zip
```

I assumed hashing it proved the original wasn't tampered with.

Correction: this gives me a starting hash to compare against later. It doesn't prove there was no tampering before I calculated it.

A supplied checksum needs a trusted source if I'm using it to check authenticity. Here I calculated my own baseline.

Then I hashed the extracted evidence itself:

```text
sha256sum temp_extract_dir/c46-FirstHack/FirstHack.ad1
130d91294aaa402a7a8770e5b81aabc76cb333019cf07a795b91cb9d4f72fe5b  temp_extract_dir/c46-FirstHack/FirstHack.ad1
```

The zip and the extracted file have separate hashes because they contain different bytes.

## What kind of file is this?

```text
file temp_extract_dir/c46-FirstHack/FirstHack.ad1
temp_extract_dir/c46-FirstHack/FirstHack.ad1: data
```

Data?

Correction: `file` didn't recognize a more specific format. That doesn't mean the file is broken, and it doesn't tell me how to open it yet.

A filename isn't enough to identify what's inside. This time `file` couldn't give me a specific answer either.

## What I learned so far

- Hashes give me a baseline to check for later changes.
- Hashing the zip doesn't give me the hash of the extracted evidence.
- `file` returning `data` means it didn't identify a specific format.

## The accompanying report

I read `FirstHack.ad1.txt`.

okay... not sure what I'm looking at yet.

Rough observations from the report:

- Created by AccessData FTK Imager 4.5.0.3.
- Custom Content Sources lists `boot`, `var/log`, and `root` from partition 5 of `Horcrux.E01`, labelled ext4.
- Acquisition and verification are dated May 25, 2021.
- The report says its MD5 and SHA1 checksums were verified. I haven't independently checked what those hashes cover.

At first I thought this might be the whole partition because those are all Linux folders.

Correction: the partition is where the files came from. The report lists selected folders, with subdirectories included. This AD1 isn't a complete copy of the disk.

Also `/` and `/root` are different. `/` is the top of the filesystem, `/root` is the root user's home folder. ext4 is the filesystem used by the source partition.

The Windows Desktop paths in the report are where the image was stored during collection. They don't make the captured system Windows.

## Opening it

I opened Exterro FTK Imager 8.3.0.27 on Windows, then File > Add Evidence Item > Image File and selected `FirstHack.ad1`.

Expanded the evidence tree and found the collected folders. FTK is a viewer here, not a terminal inside the captured machine.

## Bash history and scripts

Found `/root/.bash_history`.

okay this is actually readable. Lots of Metasploit commands, failed-looking commands, scripts, and files to follow up on.

Some of the lines:

```bash
touch snky snky > /root/Desktop/SuperSecretFile.txt
cat snky snky > /root/Desktop/SuperSecretFile.txt
cd Documents/
mkdir myfirsthack
cd myfirsthack/
touch firstscript
vim firstscript
chmod +x firstscript
./firstscript
cp firstscript firstscript_fixed
vim firstscript_fixed
./firstscript_fixed
```

I mixed up editing a script with making it executable. `vim` edits it, `chmod +x` gives execute permission, and `./firstscript` attempts to run it.

History records commands, not their results. Seeing the command doesn't prove it succeeded.

The first script had three lines:

```bash
pwd
ip route | grep default
netstat | grep 80
```

It checks the current directory, the default route, and network information filtered for `80`. No output redirection in this script, so it prints rather than saving to a file.

The fixed version added messages and highlighting:

```bash
echo "Showing you your current path"
pwd
echo "Show my default route"
ip route | grep --color default
echo "Show network connections w/ port 80"
netstat | grep --color 80
echo "Heck yeah! I can write bash too Young"
```

Correction: `grep 80` isn't a precise port-80 filter. It matches any line containing `80`, including `8080` or part of an address.

I also saw `history > history.txt` in the history, but didn't find `history.txt` in the displayed `/root/Documents/myfirsthack/` listing. That doesn't prove the command failed. It might not have stayed there.

`bob.txt` was there and 0 bytes. `touch` creates an empty file if it doesn't exist, or updates timestamps if it does. It doesn't write text into it.

## Q1 - Linux distribution

I guessed boot because the bootloader loads the OS.

Found these in `/boot`:

```text
vmlinuz-4.13.0-kali1-amd64
config-4.13.0-kali1-amd64
```

The config had a lot of `ARCH` and I wondered if that meant Arch Linux.

Correction: `ARCH` means architecture in this context. `amd64` is the processor architecture. The `kali1` part points to Kali.

Checked `/boot/grub/grub.cfg` too and saw Kali GNU/Linux entries.

Answer: **Kali Linux**.

## Q2 - MD5 of Apache access.log

Looked in `/var/log/apache2/access.log`. Blank preview, and FTK showed exactly 0 bytes.

Used Export File Hash List:

```text
MD5,SHA1
d41d8cd98f00b204e9800998ecf8427e,da39a3ee5e6b4b0d3255bfef95601890afd80709
```

Answer: **d41d8cd98f00b204e9800998ecf8427e**.

I thought an empty file's name, timestamps, and metadata would change its hash.

Correction: this hashes the file's contents. Filesystem names and timestamps aren't included. Every zero-byte file has this MD5. Metadata stored inside a file, like EXIF, does count because it's part of the bytes.

## Q3 - Downloaded credential dumping tool

Downloads seemed like the obvious place. Found `/root/Downloads/mimikatz_trunk.zip`.

Inside it were Win32 and x64 folders and a README. The README describes extracting passwords, hashes, PINs, and Kerberos tickets from memory.

I thought credential dumping might mean a collection of previously hacked passwords. It's actually extracting authentication material from a system.

Answer: **mimikatz_trunk.zip**.

The archive being present doesn't prove it ran or stole credentials. The Windows-related contents also don't change the distribution of this captured Linux system.

## Q4 - Super-secret file path

The history points to:

```text
/root/Desktop/SuperSecretFile.txt
```

I didn't see it in Desktop, but the lab accepted the path.

The commands targeted that path. I haven't established why the file isn't in the listing or whether it was later removed.

## Q5 - Program that used the JPG

I started thinking about the nearby scripts and install attempts, but the direct line was:

```bash
binwalk didyouthinkwedmakeiteasy.jpg
```

Answer: **binwalk**. A tool is still a program.

## Q6 - Third checklist goal

Found the checklist on Desktop:

```text
- Gain Bob's Trust
- Learn how to hack
- Profit
```

Answer: **Profit**.

Looks like a plan involving Bob, but the word Profit doesn't prove money was stolen.

## Q7 - How many times Apache ran

`access.log`, `error.log`, and `other_vhosts_access.log` all showed 0 bytes. I checked logs for Apache entries and didn't find any.

Tried **0**, and it was accepted.

Limitation: I didn't record a reproducible search over every available log. Empty logs and not finding an entry don't establish that Apache never ran. This is the accepted lab answer, with weaker evidence than some of the other answers.

## Q8 - File containing attack evidence

Found `/root/irZLAohL.jpeg`.

It showed a Windows command prompt in `C:\Users\Bob\AppData\Local\Temp`, an executable called `aylmao.exe`, and AlphaSOC Network Flight Simulator help text. At the bottom was `aylmao.exe run`.

Answer: **irZLAohL.jpeg**.

The screenshot is the file the question wants, not the executable shown inside it. It shows the simulator and a typed command, but doesn't independently demonstrate that an attack completed or identify this Linux machine as its source.

## Q9 - Person taunted in the script

From `firstscript_fixed`:

```bash
echo "Heck yeah! I can write bash too Young"
```

Answer: **Young**.

## Q10 - su at 11:26

Checked `/var/log/auth.log` and found repeated entries around March 20, 11:26:22-23:

```text
Mar 20 11:26:22 KarenHacker su[4060]: Successful su for postgres by root
Mar 20 11:26:22 KarenHacker su[4060]: + ??? root:postgres
Mar 20 11:26:22 KarenHacker su[4060]: pam_unix(su:session): session opened for user postgres by (uid=0)
```

Answer accepted by the lab: **postgres**.

wait, the question says someone gained root access. These lines say the opposite: root switched to postgres. `KarenHacker` is the hostname. The question and hints don't match the direction shown in these entries, so I'm keeping the discrepancy here.

## Q11 - Current working directory from history

Traced the directory changes, including:

```bash
cd ../root/Documents/myfirsthack/../../Desktop/
cd ../Documents/myfirsthack/
```

The final `pwd` has no output saved in the history, but the last recorded directory change leads to this location if it succeeded:

```text
/root/Documents/myfirsthack/
```

Answer accepted. All 11 done.

## What I learned by the end

- Start with what the evidence collection actually contains. Selected folders aren't the whole disk.
- Pick a source that fits the question: boot configuration for the distribution, Downloads for a downloaded file, auth logs for user switching.
- Find the command that actually names the file instead of connecting nearby commands automatically.
- Missing isn't the same as failed. A command in history isn't proof of successful execution.
- Read who switched to whom in authentication logs. Don't force the evidence to agree with the question.
- Keep the file path and the exact line that supports an answer, not just the answer.

## Still to practice

- Repeat one investigation without hints and explain why I chose that artifact.
- Practice a documented log search so I can explain how I reached a negative finding.
- Organize these rough notes later.
