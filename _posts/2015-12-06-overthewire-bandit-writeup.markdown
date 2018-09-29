---
layout: post
title:  "OverTheWire: Bandit Write-up"
date:   2015-12-06
---

This is a detailed write-up of my solutions to problems 0 - 25 for the [Bandit wargame](http://overthewire.org/wargames/bandit/). This wargame is aimed at beginners, each level contains a password used to gain access to the next level. The game is linear, you start at Level 0 and move up towards Level 26. All passwords have been censored out of respect for the wargame and future players.

# Level 0

Start by ssh'ing into Level 0 using the credentials provided.

{% highlight text %}
ssh bandit0@bandit.labs.overthewire.org
{% endhighlight %}

# Level 0 → Level 1

We are told that the file **readme** contains the password for Level 1. Using the <code class="inline-highlight">ls</code> command, we can see that the file exists.

To display the file, we use cat.

{% highlight text %}
cat readme
boJ9****************************
{% endhighlight %}

# Level 1 → Level 2

The password is stored in a file named '-'.

The character '-' however, is shell syntax for standard input (STDIN), so <code class="inline-highlight">cat -</code> waits for STDIN.

To get around this you can specify the path of the file using <code class="inline-highlight">./</code>

{% highlight text %}
cat ./- 
CV1D****************************
{% endhighlight %}

# Level 2 → Level 3

The password is located in a file named 'spaces in this filename'. <code class="inline-highlight">cat</code> can print out multiple files, and does so by separating files by spaces. Therefore, by doing <code class="inline-highlight">cat spaces in this filename</code>, cat will try to print out the file 'spaces' and then the file 'in' and then 'this' and then 'filename'. Which, none of those files exist. Therefore, in order to pass the entire file name as one file, we can group the name in single quotes.

{% highlight bash %}
cat 'spaces in this filename'
UmHa****************************
{% endhighlight %}

# Level 3 → Level 4

The password is stored in a hidden file in the directory 'inhere'. Navigate to this directory through <code class="inline-highlight">cd inhere/</code>. Once there, you need to use <code class="inline-highlight">ls -a</code> to display all files, included those which are hidden. The only hidden file is '.hidden' so we cat that file.

{% highlight bash %}
cat '.hidden'
pIwr****************************
{% endhighlight %}

# Level 4 → Level 5

Inside the **inhere** directory, there are several files, each starting with a dash '-', thus when you try to cat them out, cat treats the filename as an argument instead of a file. The Level 1 trick comes in handy and we can use <code class="inline-highlight">cat ./*</code> to print out each file.

We could write a small script to print out each on a  new line, but it's easy to find the password in the mess.

{% highlight text %}
koRe****************************
{% endhighlight %}

Alternatively, you can use the <code class="inline-highlight">file ./*</code> to print out which file is ASCII and then cat just that file, or use the <code class="inline-highlight">strings</code> filter to output only ASCII text from every file <code class="inline-highlight">strings ./* </code>

# Level 5 → Level 6

The file is one out of many inside anyone of the possible sub-folders!

<code class="inline-highlight">find *</code> lists all the files inside the sub-folder mess we have found. Luckily we are told some properties about the file, Most notably, it's size. <code class="inline-highlight">find</code> allows us to perform a test on each file it finds, like test for file size. Thus,

{% highlight bash %}
find * -size 1033c
{% endhighlight %}

prints out the correct file. Cat that file and we get:

{% highlight bash %}
DXjZ****************************
{% endhighlight %}

# Level 6 → Level 7

Slightly more difficult. The file is stored somewhere on the server. Navigate to / directory and perform a <code class="inline-highlight">find *</code>. This time we need to use all properties otherwise we will get more results than what we need.

{% highlight bash %}
find * -size 33c -group bandit6 -user bandit7
{% endhighlight %}

However this output prints out lots of files which we do not have permission to read, obviously not going to be the password file. Piping this input into a <code class="inline-highlight">grep</code> command to filter out those message leaves us with the file that we were searching for.

{% highlight bash %}
find * -size 33c -group bandit6 -user bandit7 2>&1 | egrep -v 'Permission denied$'
var/lib/dpkg/info/bandit7.password

cat var/lib/dpkg/info/bandit7.password
HKBP****************************
{% endhighlight %}

# Level 7 → Level 8

Since the file is incredibly large, we need to use <code class="inline-highlight">grep</code> to find the word millionth.

{% highlight bash %}
cat data.txt | grep 'millionth'
cvX2****************************
{% endhighlight %}

# Level 8 → Level 9

The password only appears once in the whole file, so we can use <code class="inline-highlight">uniq -u</code> to get unique lines. However as uniq only works on adjacent lines, we need to <code class="inline-highlight">sort</code> the file first.

{% highlight bash %}
cat data.txt | sort | uniq -u
UsvV****************************
{% endhighlight %}

# Level 9 → Level 10

In order to get human readable characters we have to use the <code class="inline-highlight">strings</code> filter, and then <code class="inline-highlight">grep</code> for lines which start with '='

{% highlight bash %}
strings data.txt | grep '^='
truK****************************
{% endhighlight %}

# Level 10 → Level 11

Base64 is used to translate all binary data into human readable ASCII. We need to decode it in order to get our password. A quick <code class="inline-highlight">man base64</code> later and...

{% highlight bash %}
base64 -d data.txt
The password is IFuk****************************
{% endhighlight %}

# Level 11 → Level 12

ROT 13 is easy to understand, a little annoying to decode using <code class="inline-highlight">tr</code>, but possible.

{% highlight bash %}
cat data.txt | tr '[N-ZA-Mn-za-m]' '[A-Za-z]'
The password is 5Te8****************************
{% endhighlight %}

# Level 12 → Level 13

A hex dump which has been repeatedly compressed. Here we go...

Using <code class="inline-highlight">xxd</code> we can reformat the hex dump into binary

{% highlight bash %}
xxd -r data.txt > data
{% endhighlight %}

Here's where the <code class="inline-highlight">file</code> filter comes in handy. <code class="inline-highlight">file data</code> shows us that it was compressed with gzip. We need to first rename the file to data.gz and then use <code class="inline-highlight">gunzip data.gz</code>

Still not plain text, <code class="inline-highlight">file data</code> shows us that it was now compressed with bzip2. bzcat unzips it to a gzip again, Now its a tar archive.

<code class="inline-highlight">tar -xvf data</code> unpacks the archive and reveals data5.bin. again another tar archive. and we get data6.bin. bzip 2 again. Oh look now another tar archive. And another gzip.

FINALLY we reach the plain text after what felt like 30 minutes.

{% highlight bash %}
The password is 8Zjy****************************
{% endhighlight %}

# Level 13 → Level 14

We are given the private key for bandit14, that's good enough for me!

Using <code class="inline-highlight">sftp</code> grab the private key and store it locally on my computer, the file needs to have its modifications changed in order to be recognized.

{% highlight bash %}
chmod 600 sshkey.private
{% endhighlight %}

Now you can use ssh.

{% highlight bash %}
ssh -i sshkey.private bandit14@bandit.labs.overthewire.org.
{% endhighlight %}

Future Note:
We don't need to use sftp to get the key, we can use ssh as user bandit13 using the following:

{% highlight bash %}
ssh -i sshkey.private bandit14@localhost
{% endhighlight %}

# Level 14 → Level 15

For this task we need the password for Level 13, now that we are bandit14 we can read the file.

{% highlight bash %}
cat /etc/bandit_pass/bandit14
4wcY****************************
{% endhighlight %}

Then, we need to submit this password to the server listening on port 30000 of localhost. netcat is our friend here.

{% highlight bash %}
nc localhost 30000
BfMY****************************
{% endhighlight %}

# Level 15 → Level 16

We are required to submit the password to a server listening on port 30001 but this time communicate using SSL encryption. Communicating using SSL requires us to use <code class="inline-highlight">openssl</code>. A quick lookup of the syntax and we get

{% highlight bash %}
openssl s_client -connect localhost:30001
{% endhighlight %}

Except this prints out the **HEARTBEATING** message and not the password. The hint says to use -quiet to suppress that message, then the password will appear.

{% highlight bash %}
openssl s_client -connect localhost:30001 -quiet
cluF****************************
{% endhighlight %}

# Level 16 → Level 17

A server is listening on a port somewhere between 30000 and 31000. We can use <code class="inline-highlight">nmap</code> to find out which ports are open and which aren't.

{% highlight bash %}
nmap localhost -p31000-32000
    31003/tcp open  unknown
    31046/tcp open  unknown
    31518/tcp open  unknown
    31691/tcp open  unknown
    31790/tcp open  unknown
    31960/tcp open  unknown
{% endhighlight %}

Some of these use SSL and some don't, quick trial and error solves this problem and we get our prize. A private key!

{% highlight bash %}
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAvmOkuifmMg6HL2YPI
....
....
....
-----END RSA PRIVATE KEY-----
{% endhighlight %}

# Level 17 → Level 18

Two files, one line of difference. The <code class="inline-highlight">diff</code> command helps us here.

{% highlight bash %}
diff passwords.new passwords.old
< kfBf****************************
---
> BS8bqB1kqkinKJjuxL6k072Qq9NRwQpR
{% endhighlight %}

And our password is,

{% highlight text %}
kfBf****************************
{% endhighlight %}

# Level 18 → Level 19

Level 18 is weird, I get logged off when I ssh in because apparently someone has modified .bashrc

Easy way around, connect via sftp and get the file. No kicking :P

{% highlight text %}
Iuek****************************
{% endhighlight %}

An alternative method is to use the option for ssh which allows a command to be executed as soon as you are logged in. Something like this:

{% highlight bash %}
ssh bandit18@bandit.labs.overthewire.org -t 'command; cat readme'
{% endhighlight %}

will also give you the answer.

# Level 19 → Level 20

We are given a **setuid binary** in order to give us permission to retrieve the password. The binary allows us to execute commands as another user (presumably bandit20) thus,

{% highlight bash %}
./bandit20-do cat /etc/bandit_pass/bandit20
{% endhighlight %}

cats out the password for level 20.

{% highlight text %}
GbKk****************************
{% endhighlight %}

# Level 20 → Level 21

This is a bit tricky. our binary will connect to a port on localhost. The hint is that we need to run two consoles. The first is use <code class="inline-highlight">nc</code> to listen to a port and submit the current password for this level.

We can do this through:

{% highlight bash %}
echo 'GbKk****************************' | nc -l 1337
{% endhighlight %}

in the other terminal we run <code class="inline-highlight">./suconnect 1337</code>

Because the passwords match, it sends the password back to the listener which prints out the new password for us.

{% highlight text %}
gE26****************************
{% endhighlight %}

# Level 21 → Level 22

The configuration file tells us that a shell program is being run and then outputted to <code class="inline-highlight">/dev/null</code>

If we navigate to and execute that script we see it echos the password into a temporary file, <code class="inline-highlight">/tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv</code>

lets cat out that file and we get our password.

{% highlight text %}
Yk7o****************************
{% endhighlight %}

# Level 22 → Level 23

This task is similar to 21 except the shell script is more involved this time

{% highlight bash %}
#!/bin/bash

myname=$(whoami)
mytarget=$(echo I am user $myname | md5sum | cut -d ' ' -f 1)

echo "Copying passwordfile /etc/bandit_pass/$myname to /tmp/$mytarget"

cat /etc/bandit_pass/$myname > /tmp/$mytarget
{% endhighlight %}

What this is doing is printing the password for Level 22, which is what we already have. We want to print out the password for level 23. Since we can't edit the file, or create a new script, we need to find out where it would place the password for bandit23

{% highlight bash %}
echo I am user bandit23 | md5sum | cut -d ' ' -f 
8ca319486bfbbc3663ea0fbe81326349
{% endhighlight %}

lucky for us, that file exists!

{% highlight bash %}
cat /tmp/8ca319486bfbbc3663ea0fbe81326349
the password is jc1u****************************
{% endhighlight %}

# Level 23 → Level 24

Similar challenge, let's find the script <code class="inline-highlight">/usr/bin/cronjob_bandit24.sh</code>. This time we get more complex

{% highlight bash %}
#!/bin/bash

myname=$(whoami)
cd /var/spool/$myname
echo "Executing and deleting all scripts in /var/spool/$myname:"

for i in * .*;
do
   if [ "$i" != "." -a "$i" != ".." ];   then
      echo "Handling $i"
      timeout -s 9 60 "./$i"
      rm -f "./$i"
  fi
done
{% endhighlight %}

This doesn't have anything to do with passwords… but, It says it is executing and then deleting scripts in <code class="inline-highlight">/var/spool/$myname</code>, so maybe we can write a script inside there to be run and get our password.

The previous script for level 23 will be what we want, just need to make some modifications

{% highlight bash %}
cp /usr/bin/cronjob_bandit23.sh /var/spool/bandit24/test.sh
{% endhighlight %}

Edit the file and modify <code class="inline-highlight">whoami</code> into bandit 24 and run it

Our password has been copied into <code class="inline-highlight">/tmp/ee4ee1703b083edac9f8183e4ae70293</code>

and we’ve got it.

{% highlight text %}
UoMY****************************
{% endhighlight %}

# Level 24 → Level 25

There's a daemon waiting for the password and a 4 digit code in order to let us pass.

Let's create a shell script to brute force this pin

{% highlight bash %}
for pin in {0..9}{0..9}{0..9}{0..9}; do
   echo UoMY**************************** $pin | nc localhost 30002 
done
{% endhighlight %}

Letting this run for a while we find that 5669 is the pin and our password is:

{% highlight text %}
uNG9****************************
{% endhighlight %}

# Level 25 → Level 26

Once we login to 25 we see a private key for bandit26, save this for later.

We need to find out which Shell bandit26 is using. From looking at <code class="inline-highlight">/etc/passwd</code> it seems that bandit26 uses <code class="inline-highlight">/usr/bin/showtext</code> as the default shell.

All this shell does is <code class="inline-highlight">more text.txt</code> and then exits. We need to find a way around this..

In order for <code class="inline-highlight">more</code> to activate we need to re-size the window to be quite small. only a few lines. More allows for interactivity, somehow we need to use this. More also allows for commands to be run with !, but this didn't seem to work. We can also open <code class="inline-highlight">vi</code> on the file which is handy. From vi we can open another file, like say,

{% highlight text %}
/etc/bandit_pass/bandit26
{% endhighlight %}

and there we go, our password

{% highlight text %}
5czg****************************
{% endhighlight %}

This next part is tricky, I had to look this up, but apparently we need to set the shell and then run it through vi. Whilst in vi do: 

{% highlight text %}
:set shell sh=/bin/bash
:sh
{% endhighlight %}

Now we can look around!
cat out <code class="inline-highlight">README.txt</code> and we have finished the wargame!

{% highlight text %}
cat README.txt
Congratulations on solving the last level of this game!
At this moment, there are no more levels to play in this game. However, we are constantly working
on new levels and will most likely expand this game with more levels soon.
Keep an eye out for an announcement on our usual communication channels!
In the meantime, you could play some of our other wargames.
If you have an idea for an awesome new level, please let us know!
{% endhighlight %}
