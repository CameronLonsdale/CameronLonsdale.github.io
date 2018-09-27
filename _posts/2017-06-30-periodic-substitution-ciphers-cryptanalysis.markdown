---
layout: post
title:  "Periodic Substitution Cipher Automated Cryptanalysis"
date:   2017-06-30
---

Periodic Substitution Ciphers are a type of polyalphabetic substitution cipher where the key sequence is repeated after a period of p plaintext symbols. The encryption transformation can be expressed as a set of mapping functions corresponding to the p different keys $K = k_{1}, ..., k_{p}$:

<center>
$$f_{i}: M \rightarrow C_{i}, for \ 1 \leq i \leq p$$

$$M = \{A, B, C, ..., X, Y, Z\}$$ (plaintext alphabet)
<br><br>
$$C_{1} = \{V, G, D, ..., C, X, E\}$$ (first key)

$$\vdots$$

$$C_{p} = \{K, F, M, ..., R, W, S\}$$ (p<sup>th</sup> key)
</center>
<br>

# Experimentation in Cryptanalysis

From experience cracking monoalphabetic substitution ciphers, I predict that as the length of ciphertext decreases the cipher will become more difficult to break to the point of automation failing to produce any recognizable decryption. As I will be using a randomized hill climbing method, I predict that the larger the period the more difficult it will be to break automatically. This will be due to amount of work required to break the ciphertext increases as more keys are needed.

## Ciphertext Length - Results

<img src="{{ site.baseurl }}/assets/img/periodic-substitution-cipher/chart1.png">

Surprisingly, the correct decryption was found up until the 804 mark by my algorithm. The difference between the blue line and the red line indicates that my detection algorithm needs more work, because I was able to select correct solutions when the computer was unable to. The overall trend confirms my hypothesis, the shorter the text the less likely correct decryption will be found. Since the code is a hill climb, sufficient data is needed to guide the climber into the maximum node, without that it will end up somewhere wrong.


## Key Period - Results

<img src="{{ site.baseurl }}/assets/img/periodic-substitution-cipher/chart2.png">

As expected, the greater the period the increasingly difficult it becomes to detect. From the orange data lines its evident that you can improve the decryption results by increasing the number of steps taken during the hill climb. The larger the period the larger the steps. Though this does show an improvement from the blue (which is the same number of steps for each period), the orange is still trending downwards, indicating the probability will still eventually win over the hill climb in the long run.

Why? Well when you consider that essentially a hill climb is just a guided brute force that uses randomness and think about how adding more alphabets increases the keyspace, It's clear that higher periods require more work and more luck.

$P(key)  = \dfrac{1}{(26!)^{p}}$. As the period (p) increases, the number of ways to pick a key increases exponentially, the probability of finding that key decreasing in the same rate.

Though the orange line trends down, for the sample periods chosen, the first sentence of the best result usually was similar enough to determine the source of the entire text.

# Appendix

* Percentage of correct decryption was calculated by comparing the hamming distance of the plaintext and the best scoring decrypted ciphertext.
* This was not very scientific.

Here is the code for solving periodic substitution ciphers. Please checkout [lantern](https://github.com/CameronLonsdale/lantern)!

<script src="https://gist.github.com/CameronLonsdale/d8ea9d199b057125bf9ed95a59802354.js"></script>
