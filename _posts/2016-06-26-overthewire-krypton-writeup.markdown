---
layout: post
title:  "OverTheWire: Krypton Write-up"
date:   2016-06-26
---


The [Krypton wargame](http://overthewire.org/wargames/krypton/) is all about cryptography, each challenge you are given an encrypted password and have to break it using a different technique. This write up will not post any of the passwords out of respect for the wargame and future players.

 
<center>
2018 Update: You can now solve 5/6 of this wargame in roughly 60 seconds using Torch
</center>


# Level 0

As a big hint, the game tells us that the password is encoded in base64, which is a common binary to text encoding that makes use of 64 printable ASCII characters.

{% highlight text %}
S1JZUFRPTklTR1JFQVQ=
{% endhighlight %}

The easiest way to decode this is to use the unix filter <code class="inline-highlight">base64 -d</code>

{% highlight text %}
$ base64 -d
S1JZUFRPTklTR1JFQVQ=
KRYPTON*******
{% endhighlight %}

# Level 1

We are told the password in this level is encrypted using a very simple letter rotation. In fact, it is ROT13, where each letter has been swapped with the letter 13 places ahead of it. The mapping looks something like this.

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/33/ROT13_table_with_example.svg/640px-ROT13_table_with_example.svg.png">

We can simply reverse this mapping to give us the plain text. Using the translate unix filter, we can easily define this relation and apply it to the string.

{% highlight bash %}
cat krypton2 | tr '[A-Za-z]' '[N-ZA-Mn-za-m]'
LEVEL TWO PASSWORD ******
{% endhighlight %}

# Level 2

We are told that the crypto is a Caesar cipher, but we do not know what rotation it is. We could simply brute force all 24 possibilities but we are given an executable binary which will encrypt any plain text for us using the same Caesar cipher.

This is known as a chosen-plaintext attack, by simply using the binary to encrypt the entire English alphabet, we can reverse engineer the cipher and decrypt the message.

{% highlight text %}
abcdefghijklmnopqrstuvwxyz
MNOPQRSTUVWXYZABCDEFGHIJKL
{% endhighlight %}

With a quick calculation, we can see the rotation is 12, so reversing the password back by 12 (or forward by 14) will give us our password.

{% highlight text %}
CAESAR******
{% endhighlight %}

# Level 3

In this level, we are told the cipher is a substitution cipher, but we do not have access to an executable. What we do have access to is over 4000 characters of encrypted text. Using a technique known as frequency analysis, we can attempt to crack what the substitution is. As ‘e’ is the most common letter used in the English alphabet, whatever the most common letter is in the found text, is most likely ‘e’ in plain-text. Following this logic and using a larger frequency table to reverse map letters, we can start to see words forming. The attack model being used here is a known-plaintext attack.

Using the help of analysis websites we can see that the entire alphabet has been mapped to this.

{% highlight text %}
abcdefghijklmnopqrstuvwxyz
qazwsxedcrfvtgbyhnujmikolp
{% endhighlight %}

Applying this mapping to our encrypted password and we get

{% highlight text %}
WELL DONE THE LEVEL FOUR PASSWORD IS *****
{% endhighlight %}

# Level 4

Out with the mono-alphabetic ciphers and in with the poly-alphabetic ones. A poly-alphabetic cipher is based on substitution but using multiple alphabets instead of just one. The classic cipher, and the one used in this level is a Vigenère cipher. It works in this way.

with our key, <code class="inline-highlight">K = GOLD</code> and plaintext <code class="inline-highlight">P = PROCEED MEETING AS AGREED</code> then the encryption is as follows.

{% highlight text %}
P |  P R O C E E D M E E T I N G A S A G R E E D
K |  G O L D G O L D G O L D G O L D G O L D G O

P |  15 17 14 2 4 4 3 12 4 4 19 8 13 6 0 18 0 6 17 4 4 3
K |  6 14 11 3 6 14 11 3 6 14 11 3 6 14 11 3 6 14 11 3 6 14
C |  21 5 25 5 10 18 14 15 10 18 4 11 19 20 11 21 6 20 2 8 10 17
{% endhighlight %}

Which produces  a cipher text of <code class="inline-highlight">VFZFK SOPKS ELTUL VGUCH KR.</code>

In order to break this cipher we can use frequency analysis. We are given the length of the key K, which is 6. What we can try is breaking up the cipher text into groups of 6 characters and performing analysis on the first character of each group, to try and get the first character of the key. Then repeat this process for each subsequent character in the key. With the help of an analysis website, we see a possible key could be <code class="inline-highlight">FREKEY</code>. Luckily, this is the key and can be used to decrypt the password for level 5.

# Level 5

Another Vigenère cipher, except this time we do not know the length of the key. Breaking this requires frequency analysis and abit of intuition. Using the same analysis website, we can perform statistical analysis and the website provides a few keys which could be correct.

One of them is <code class="inline-highlight">KEYZEBGTH</code>, using this the found text decrypts partially to <code class="inline-highlight">IT WMS FHE BEST AF FIMESA</code>. Intuitively, this looks as though its meant to say <code class="inline-highlight">IT WAS THE BEST OF TIMES</code>. Modifying our key to fit this plaintext, our key transforms into <code class="inline-highlight">KEYLENGTH</code>, which is the correct key to decrypt the password for level 6

# Level 6

Coming soon