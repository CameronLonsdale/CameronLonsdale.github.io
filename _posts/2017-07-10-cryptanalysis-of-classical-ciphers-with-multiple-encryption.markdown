---
layout: post
title:  "Cryptanalysis of Classical Ciphers with Multiple Encryption"
date:   2017-07-10
---

Have you ever wondered how [multiple encryption](https://en.wikipedia.org/wiki/Multiple_encryption) affects the security of the famous classical ciphers, Caesar, Substitution and Vigenère? Does encrypting plaintext multiple times improve, weaken or not affect the overall Cipher strength? The conclusion, unsurprisingly, is that multiple rounds of encryption are equivalent to encrypting the plaintext once. Here’s why.

# Caesar Cipher

The [Caesar Cipher](https://en.wikipedia.org/wiki/Caesar_cipher) is a shift cipher where letters are replaced by those _k_ along in the alphabet. When shifting our plaintext <code class="inline-highlight">EXAMPLE</code> by _5_ we get <code class="inline-highlight">JCFRUQJ</code>. Shifting this ciphertext by _3_ results in <code class="inline-highlight">MFIUXTM</code>. With the help of [lantern](https://github.com/CameronLonsdale/lantern) this ciphertext can be decrypted.

{% highlight python %}
from lantern import caesar
decryptions = caesar.crack("MFIUXTM", fitness.english.quadgrams)
print(decryptions[0])
EXAMPLE

print(decryptions[0].key)
8
{% endhighlight %}

The plaintext was successfully found by brute forcing keys between 0 and 25. The encryption key for this text was _8_. Relating this back to our initial two encryption keys, 3 and 5, it is clear that 8 is the result of combining the two shifts together. No matter how many times the text is shifted it will always stay within the bounds of the alphabet and the resulting shift can be represented by a single value. Hence, multiple encryption of the Caesar Cipher is equivalent to a single encryption, meaning it can be broken using standard methods.

# Substitution Cipher

The [Substitution Cipher](https://en.wikipedia.org/wiki/Substitution_cipher) extends the Caesar Cipher by allowing arbitrary letter replacements instead of shifting. Single encryption can be easily broken using letter frequency and dependencies.

{% highlight text %}
ITISALONGESTABLISHEDFACTTHATAREADERWILLBEDISTRACTEDBYTHEREADABLECONTENTOFAPAGEWHENLOOKINGATITSLAYOUTTHEPOINTOFUSINGLOREMIPSUMISTHATITHASAMOREORLESSNORMALDISTRIBUTIONOFLETTERSASOPPOSEDTOUSINGCONTENTHERECONTENTHEREMAKINGITLOOKLIKEREADABLEENGLISHMANYDESKTOPPUBLISHINGPACKAGESANDWEBPAGEEDITORSNOWUSELOREMIPSUMASTHEIRDEFAULTMODELTEXTANDASEARCHFORLOREMIPSUMWILLUNCOVERMANYWEBSITESSTILLINTHEIRINFANCYVARIOUSVERSIONSHAVEEVOLVEDOVERTHEYEARSSOMETIMESBYACCIDENTSOMETIMESONPURPOSEINJECTEDHUMOURANDTHELIKE
{% endhighlight %}

Encrypt using the key: <code class="inline-highlight">VORJLFYIDUMQTNHGEWAXBCPSZK</code>
{% highlight text %}
DXDAVQHNYLAXVOQDAILJFVRXXIVXVWLVJLWPDQQOLJDAXWVRXLJOZXILWLVJVOQLRHNXLNXHFVGVYLPILNQHHMDNYVXDXAQVZHBXXILGHDNXHFBADNYQHWLTDGABTDAXIVXDXIVAVTHWLHWQLAANHWTVQJDAXWDOBXDHNHFQLXXLWAVAHGGHALJXHBADNYRHNXLNXILWLRHNXLNXILWLTVMDNYDXQHHMQDMLWLVJVOQLLNYQDAITVNZJLAMXHGGBOQDAIDNYGVRMVYLAVNJPLOGVYLLJDXHWANHPBALQHWLTDGABTVAXILDWJLFVBQXTHJLQXLSXVNJVALVWRIFHWQHWLTDGABTPDQQBNRHCLWTVNZPLOADXLAAXDQQDNXILDWDNFVNRZCVWDHBACLWADHNAIVCLLCHQCLJHCLWXILZLVWAAHTLXDTLAOZVRRDJLNXAHTLXDTLAHNGBWGHALDNULRXLJIBTHBWVNJXILQDML
{% endhighlight %}

Encrypt again using key: <code class="inline-highlight">SAEYOCBPRNZKVGJQFXMILDTWUH</code>
{% highlight text %}
YWYSDFPGUKSWDJFYSRKNCDXWWRDWDTKDNKTQYFFJKNYSWTDXWKNJHWRKTKDNDJFKXPGWKGWPCDBDUKQRKGFPPVYGUDWYWSFDHPAWWRKBPYGWPCASYGUFPTKIYBSAIYSWRDWYWRDSDIPTKPTFKSSGPTIDFNYSWTYJAWYPGPCFKWWKTSDSPBBPSKNWPASYGUXPGWKGWRKTKXPGWKGWRKTKIDVYGUYWFPPVFYVKTKDNDJFKKGUFYSRIDGHNKSVWPBBAJFYSRYGUBDXVDUKSDGNQKJBDUKKNYWPTSGPQASKFPTKIYBSAIDSWRKYTNKCDAFWIPNKFWKMWDGNDSKDTXRCPTFPTKIYBSAIQYFFAGXPEKTIDGHQKJSYWKSSWYFFYGWRKYTYGCDGXHEDTYPASEKTSYPGSRDEKKEPFEKNPEKTWRKHKDTSSPIKWYIKSJHDXXYNKGWSPIKWYIKSPGBATBPSKYGLKXWKNRAIPATDGNWRKFYVK
{% endhighlight %}

Since the frequency of symbols does not change with multiple encryption, we should be able to decrypt double enciphered text easily.

{% highlight python %}
from lantern import simplesubstitution
decryptions = simplesubstitution.crack(ciphertext, fitness.english.quadgrams)
print(decryptions[0])
ITISALONGESTABLISHEDFACTTHATAREADERWILLBEDISTRACTEDBYTHEREADABLECONTENTOFAPAGEWHENLOOKINGATITSLAYOUTTHEPOINTOFUSINGLOREMIPSUMISTHATITHASAMOREORLESSNORMALDISTRIBUTIONOFLETTERSASOPPOSEDTOUSINGCONTENTHERECONTENTHEREMAKINGITLOOKLIKEREADABLEENGLISHMANYDESKTOPPUBLISHINGPACKAGESANDWEBPAGEEDITORSNOWUSELOREMIPSUMASTHEIRDEFAULTMODELTEXTANDASEARCHFORLOREMIPSUMWILLUNCOVERMANYWEBSITESSTILLINTHEIRINFANCYVARIOUSVERSIONSHAVEEVOLVEDOVERTHEYEARSSOMETIMESBYACCIDENTSOMETIMESONPURPOSEINJECTEDHUMOURANDTHELIKE

print(decryptions[0].key)
DJXNKCURYLVFIGPBZTSWAEQMHO
{% endhighlight %}

The plaintext was found, our prediction was correct. Since only the symbols change and not their frequency, multiple rounds of encryption is equivalent to a single round with a different key. The final key can be determined by encrypting the key from the previous round using the key of the current round and continuing until completion. In our example above, <code class="inline-highlight">VORJLFYIDUMQTNHGEWAXBCPSZK</code> encrypted using <code class="inline-highlight">SAEYOCBPRNZKVGJQFXMILDTWUH</code> results in <code class="inline-highlight">DJXNKCURYLVFIGPBOTSWAEQMHZ</code>, our final key.

# Vigenère Cipher

The [Vigenère Cipher](https://en.wikipedia.org/wiki/Vigen%C3%A8re_cipher) is a periodic polyalphabetic substitution cipher, or in other words, it uses _p_ many values to shift the plaintext, the same shift repeated every _p_ letters. Since the frequency of symbols change with encryption it makes the ciphertext slightly harder to decrypt than Substitution. Once the period is known however, it is simply a matter of solving p many Caesar Ciphers. With multiple encryption there can be two variants, where all keys have the same period, and where the periods differ.

## Same Period

{% highlight text %}
ITISALONGESTABLISHEDFACTTHATAREADERWILLBEDISTRACTEDBYTHEREADABLECONTENTOFAPAGEWHENLOOKINGATITSLAYOUTTHEPOINTOFUSINGLOREMIPSUMISTHATITHASAMOREORLESSNORMALDISTRIBUTIONOFLETTERSASOPPOSEDTOUSINGCONTENTHERECONTENTHEREMAKINGITLOOKLIKEREADABLEENGLISHMANYDESKTOPPUBLISHINGPACKAGESANDWEBPAGEEDITORSNOWUSELOREMIPSUMASTHEIRDEFAULTMODELTEXTANDASEARCHFORLOREMIPSUMWILLUNCOVERMANYWEBSITESSTILLINTHEIRINFANCYVARIOUSVERSIONSHAVEEVOLVEDOVERTHEYEARSSOMETIMESBYACCIDENTSOMETIMESONPURPOSEINJECTEDHUMOURANDTHELIKE
{% endhighlight %}

Encrypt using the key: <code class="inline-highlight">VXUH</code>
{% highlight text %}
DQCZVIIUBBMAVYFPNEYKAXWAOEUAVOYHYBLDDIFIZACZOOUJOBXITQBLMBUKVYFLXLHAZKNVAXJHBBQOZKFVJHCUBXNPOPFHTLOAOEYWJFHAJCOZDKASJOYTDMMBHFMACXNPOEUZVJIYZLLSZPMUJOGHGACZOOCIPQCVILZSZQNLMPUZJMJVNBXAJRMPIDWVIQYUOEYYZZIUOBHACBLLHXEPIDCAGLIRGFELMBUKVYFLZKASDPBTVKSKZPEAJMJBWICZCFHNKXWRVDYZVKXDZYJHBBYKDQIYNKIDPPYSJOYTDMMBHXMACBCYYBZHPINTJAYSOBRAVKXHNBUYXEZVMIIYZJCWNRGDDIFBIZICZOGHIVQLWPCAZPMADIFPIQBLDOCUAXHJTSUYDLOZQBLZDLHZCXPLZSISQBXVQBLACBSLVOMZJJYADJYZWVUJXFXLIQMVHBNPHBMVIMOYKLMLDKDLXQYKCRGVPOUUYQBLGFEL
{% endhighlight %}

Encrypt again using key: <code class="inline-highlight">VTPM</code>
{% highlight text %}
YJRLQBXGWUBMQRUBIXNWVQLMJXJMQHNTTUAPYBUUUTRLJHJVJUMUOJQXHUJWQRUXSEWMUDCHVQYTWUFAUDUHEARGWQCBJIUTOEDMJXNIEYWMEVDLYDPEEHNFYFBNCYBMXQCBJXJLQCXKUEAEUIBGEHVTBTRLJHRUKJRHDEOEUJCXHIJLEFYHIUMMEKBBDWLHDJNGJXNKUSXGJUWMXUAXCQTBDWRMBEXDBYTXHUJWQRUXUDPEYIQFQDHWUITMEFYNRBRLXYWZFQLDQWNLQDMPURYTWUNWYJXKIDXPKINEEHNFYFBNCQBMXURKTUOTKBCFETNEJUGMQDMTIUJKSXOHHBXKUCRIIKVPYBUNDSXOUHVTDOFXRIRMUIBMYBUBDJQXYHRGVQWVOLJKYEDLLUALYEWLXQEXULXELUMHLUAMXUHXQHBLECNMYCNLROJVSYMXDJBHCUCBCUBHDFDKFEBXYDSXSJNWXKVHKHJGTJQXBYTX
{% endhighlight %}

{% highlight python %}
from lantern import vigenere
decryptions = vigenere.crack(ciphertext, fitness.ChiSquared(frequency.english.unigrams), fitness.english.quadgrams)
print(decryptions[0])
ITISALONGESTABLISHEDFACTTHATAREADERWILLBEDISTRACTEDBYTHEREADABLECONTENTOFAPAGEWHENLOOKINGATITSLAYOUTTHEPOINTOFUSINGLOREMIPSUMISTHATITHASAMOREORLESSNORMALDISTRIBUTIONOFLETTERSASOPPOSEDTOUSINGCONTENTHERECONTENTHEREMAKINGITLOOKLIKEREADABLEENGLISHMANYDESKTOPPUBLISHINGPACKAGESANDWEBPAGEEDITORSNOWUSELOREMIPSUMASTHEIRDEFAULTMODELTEXTANDASEARCHFORLOREMIPSUMWILLUNCOVERMANYWEBSITESSTILLINTHEIRINFANCYVARIOUSVERSIONSHAVEEVOLVEDOVERTHEYEARSSOMETIMESBYACCIDENTSOMETIMESONPURPOSEINJECTEDHUMOURANDTHELIKE

print(decryptions[0].key)
BLSO
{% endhighlight %}

lantern successfully decrypted the multiple encrypted ciphertext with two keys of the same period. The period was found using standard methods and the entire text could be decrypted by solving _p_ many Caesar Shifts. The final shift, represented by <code class="inline-highlight">BLSO</code> is the result of combing the shifts of <code class="inline-highlight">VXUH</code> and <code class="inline-highlight">VTPM</code>. This is intuitive to see using our understanding of how multiple encryption affects the key of a Caesar Cipher.

## Different Periods

As with the same period, different period multiple encryption is equivalent to encrypting the plaintext once. The keys are combined in the same way, the period of the final key however will vary.

The periods of the keys can have common factors or they can be [Relatively prime](https://en.wikipedia.org/wiki/Coprime_integers) (numbers where the only positive common factor is 1). Either way, the resulting period of the final encryption key is determined by the [least common multiple](https://en.wikipedia.org/wiki/Least_common_multiple) of all the key periods.

When encrypting using <code class="inline-highlight">DMZU</code> then <code class="inline-highlight">XG</code> results in the key <code class="inline-highlight">AWSA</code>. This is easy to see as <code class="inline-highlight">XG</code> can become <code class="inline-highlight">XGXG</code> without changing the encryption, we then have a multiple encryption with the same period. The least common multiple of 4 and 2 is 4, which is the length of our final key. <code class="inline-highlight">AWSA</code> can be determined by encrypting <code class="inline-highlight">DMZU</code> with the key <code class="inline-highlight">XGXG</code>.

When encrypting with <code class="inline-highlight">HJC</code> then <code class="inline-highlight">SLOZ</code> the final key is <code class="inline-highlight">ZUQGBNVIUSXB</code>. The least common multiple of 3 and 4 is 12. This can be understood visually if you imagine the keys <code class="inline-highlight">HJC</code> and <code class="inline-highlight">SLOZ</code> overlapping as they encrypt a piece of text. <code class="inline-highlight">SLOZ</code> is not a multiple of 3, hence multiple repetitions of the key are needed until we reach a length where both keys start again from the same letter. When they start again, we have a repeating pattern that represents the key that can express the multiple encryption as a single encryption.

<div class="highlight">
<pre>
| <span style="color: #99cc00">H   J   C </span>| <span style="color: #99cc00">H   J   C</span> | <span style="color: #99cc00">H   J   C</span> | <span style="color: #99cc00">H   J   C</span> |
| <span style="color: #ff9900">S   L   O   Z</span> | <span style="color: #ff9900">S   L   O   Z</span> | <span style="color: #ff9900">S   L   O   Z</span> |
</pre>
</div>

This pattern will repeat every _12_ letters, hence the encryption from plaintext to final ciphertext can be represented by a single encryption using a key of period _12_. The resultant key <code class="inline-highlight">ZUQGBNVIUSXB</code> is found by encrypting <code class="inline-highlight">HJCHJCHJCHJC</code> with the key <code class="inline-highlight">SLOZSLOZSLOZ</code>.
