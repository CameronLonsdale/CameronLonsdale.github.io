---
layout: post
title:  "OverTheWire Leviathan: Write-up"
date:   2016-07-03
---

This is a detailed write-up of my solutions to problems 0 – 6 for the [Leviathan wargame](http://overthewire.org/wargames/leviathan/), the next difficulty up from [Bandit](https://cameronlonsdale.wordpress.com/2015/12/06/overthewire-bandit-write-up/). All passwords have been censored out of respect for the wargame and future players.

# Level 0 → Level 1

Leviathan starts off very strange, there are no instructions to progress from one level to another, you just need to find the next password. The first thing we try once we have a terminal is use <code class="inline-highlight">ls</code> to view any folders / files, but we get no results. A more exhaustive listing using

{% highlight bash %}
ls -a
{% endhighlight %}

shows a hidden folder called **.backup**. Inside this folder is a bookmarks html file which is 1000+ lines long, too long to manually sift through. So, let’s use <code class="inline-highlight">grep</code> to search for a likely keyword like ‘leviathan’.

{% highlight bash %}
cat ./backup/bookmarks.html | grep 'leviathan'
the password for leviathan1 is **********
{% endhighlight %}

# Level 1 → Level 2

The only file in our home directory is a binary executable named **./check**. It asks for a password, but the current password doesn’t seem to work.

{% highlight bash %}
strings ./check
{% endhighlight %}

Reveals several pieces of information. It was a C file, includes a 'strcmp' and for some reason a '/bin/sh'.

We can use the tool <code class="inline-highlight">ltrace</code> to see what library calls happen inside **./check**, primarily, that strcmp.

{% highlight bash %}
ltrace ./check
password: test
strcmp("test", "sex")
{% endhighlight %}

Among it’s output was this line, comparing our inputted password to the word ‘sex’. Using the password ‘sex’ we are given a shell by the program.

{% highlight bash %}
whoami
leviathan2
{% endhighlight %}

With our new found permissions we can go ahead and access the password file for leviathan2.

{% highlight bash %}
cat /etc/leviathan_pass/leviathan2
**********
{% endhighlight %}

# Level 2 → Level 3

Inside our initial working directory is a binary called **./printfile**. Using a temp file with sample text, we can see that this binary simply prints out the contents of the file passed as the parameter. However, if we try to pass it /etc/leviathan_pass/leviathan3 we get the error message  “You cant have that file…”.

<code class="inline-highlight">ltrace</code> helps shed light on what is happening in both test cases.

{% highlight bash %}
ltrace ./printfile /tmp/tmp.lY0AWlFxHU/text
> access("/tmp/tmp.lY0AWlFxHU/text", 4) = 0
system("/bin/cat /tmp/tmp.lY0AWlFxHU/text")

ltrace ./printfile /etc/leviathan_pass/leviathan3
> access("/etc/leviathan_pass/leviathan3", 4) = -1
puts("You cant have that file...")
{% endhighlight %}

Our goal here is to pass in a file which access() says can be read, and then exploit the system call to /bin/cat to end up printing out the password file. The vulnerability we need to exploit is that cat will print multiple files when separated  by a space, like so.

{% highlight bash %}
cat firstFile secondFile
{% endhighlight %}

And since no filtering is happening on our input path to cat, we can provide a path that has spaces in the name.

{% highlight bash %}
mktemp -d
ln -s /etc/leviathan_pass/leviathan3 password
touch 'password foo'
./printfile 'password foo'
{% endhighlight %}

In our own temp directory, create a symbolic link to the leviathan3 password file and assign an alias name to it. Then, we must create a new file which contains the name of the symbolic link and some other garbage text. When we pass ./printfile ‘password foo’, access() will see that ‘password foo’ exists and can be read from. This then gets passed to cat as

{% highlight bash %}
/bin/cat password foo
**********
{% endhighlight %}

which will print out the password file.

# Level 3 → Level 4

The binary executable in this level is called **./level3**, its similar to  Level 1 → Level 2, in that it asks for a password. Using <code class="inline-highlight">ltrace</code> reveals the strcmp call in the code.

{% highlight C %}
strcmp("test\n", "snlprintf\n")
{% endhighlight %}

Thus, when we use the password ‘snlprintf’, the program gives us shell access. As we are leviathan4, we can go ahead and print out the password file.

{% highlight bash %}
cat /etc/leviathan_pass/leviathan4
**********
{% endhighlight %}

# Level 4 → Level 5

There is a hidden folder in our home directory, which contains binary named ./bin. When executed, it outputs some binary code. Using a binary to ASCII translator tool on the web, it becomes our password for leviathan5.

{% highlight text %}
./bin
01010100 01101001 01110100 01101000 00110100 
01100011 01101111 01101011 01100101 01101001 0000101
{% endhighlight %}

# Level 5 → Level 6

The binary in the home directory, **./leviathan5**, looks like it simply cats out the contents of the path /tmp/file.log. Creating this file with sample text confirms this.

What we learnt in  Level 2 → Level 3 about symbolic links will come in handy here. If we create a symbolic link for /etc/leviathan_pass/leviathan6 alias it to the path /tmp/file.log, then our binary will print out the password for us.

{% highlight bash %}
ln -s /etc/leviathan_pass/leviathan6 /tmp/file.log

./leviathan5
**********
{% endhighlight %}

# Level 6 → Level 7

The binary this time accepts a 4 digit code as a parameter, the assumption being, you guess the correct password and you either get the password, or privilege escalation.

We completed a similar challenge to this in Bandit, so "[It’s definitely necessary to break out vim and modify that bash script](https://www.getyarn.io/yarn-clip/d5cd2969-7efd-4b45-9bc0-d1228fbe82b3)".

{% highlight bash %}
for code in {0..9}{0..9}{0..9}{0..9}; do
    ./leviathan6 $code
done
{% endhighlight %}

Once the correct pin has been entered, we are granted leviathan7 shell access, which we then use to cat out /etc/leviathan_pass/leviathan7.

# Reflection

The most important lesson from this wargame was that in real life situations, there are no hints or instructions, you need to investigate and determine the vulnerability & exploit yourself. Additionally, we were introduced to the power of privilege escalation and to the new bash filters of <code class="inline-highlight">ltrace</code> and symbolic links: <code class="inline-highlight">ln</code>.


