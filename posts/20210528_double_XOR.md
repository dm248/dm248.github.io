---
usemathjax: true
---
# Double XOR

##### May 28, 2021

If you were wondering about how to solve the **Double XOR** challenge
at [b01lers CTF 2021](https://ctftime.org/event/1259),
then the wait is over. The solver was ready ages ago (to field the problem
I obviously had to break it first) but it took a looong while to finally type it up :) 
Like almost always, the actual thought process leading to the solution
had its twists and turns - what 
you will find below is a reasonably rectified path.

The challenge was to decrypt [this](https://github.com/b01lers/b01lers-ctf-2021/blob/main/crypto/XOR2/dist/chal2.enc)
specific ciphertext. So you had about 2700 bytes to work with. The
challenge repository can be found [here](https://github.com/b01lers/b01lers-ctf-2021/tree/main/crypto/XOR2).


### Breaking single XOR

A natural stepping stone is breaking the usual [XOR 
encryption](https://en.wikipedia.org/wiki/XOR_cipher). This is a standard task that you can learn, 
e.g., by doing the problems on [cryptopals.com](https://cryptopals.com) (we also had this on [b01lers 
CTF bootcamp](https://ctftime.org/event/1089) last August). 

#### Algorithm

Following the technique there, first you 
determine the key length by computing the average [Hamming 
distance](https://en.wikipedia.org/wiki/Hamming_distance) of ciphertext (ctxt) bytes a distance $$d$$ 
apart. Given enough ctxt, this as a function of $$d$$ will have local minima at distances that are 
multiples of the key length. Armed with the key length $$N$$, you then find each key *byte* by taking 
the set $$S(k)$$ of ctxt bytes at fixed position $$k$$ modulo $$N$$, and doing an exhaustive search (256 tries) for 
which byte value makes the frequency distribution of bytes in the given $$S(k)$$ 
the most text-like upon XOR. 
You can do that in 
sophisticated ways, such as taking the deviation in some normed space from the character distribution 
expected for English text, or maximizing likelihood based on character frequencies. But something as simple as 
scoring characters according to their category, favoring frequent types and penalizing against 
unprintables works fine too (the solver at bootcamp used space = 2, letter = 2, number = 1, printable = 1, tab/newline = 0, 
unprintable = -10). The method is pretty solid if you have 30-40 ctxt bytes per key byte or more.

#### Why it works

It is helpful to analyze what makes this all work. The Hamming distance 
inherently involves XOR because for two bytes $$a$$ and $$b$$ it is just 
the number of 1 bits in $$a \hat{} b$$.
When $$d$$ is a multiple of $$N$$, 
then 

$$c_i \hat{} c_{i+d} = p_i \hat{} p_{i+d} \ , $$

i.e., the result is then a property of the plaintext (ptxt) only because the particular key byte drops out. 
Of course, the key can be anything. If we drew key bytes from a super
limited pool of {0x00, 0x01}, then guessing the wrong $$d$$ would have a rather small effect
(which is still measurable, given enough ctxt, primarily because a 0x01 swaps spaces, A-s, and a-s with 
much less frequent !-s, @-s, and `-s). 
But for uniformly random key bytes, a wrong guess is 
noticeable with high probability:
we have on average a 50% chance of flipping bit 7, which is practically always 0 for English text - 
so in the Hamming distance we get a +0.5 increase right away.

In fact, if we know the key length, then we can also get each key byte bit-by-bit,
by comparing the likelihood of a given bit being 1 in each set $$S(k)$$ to values expected for English text:

```python
P0 = 0.452 
P1 = 0.335
P2 = 0.480
P3 = 0.357
P4 = 0.247
P5 = 0.920
P6 = 0.754
P7 = 0.025
```

(obviously the numbers here depend on the body of text used to compute those but
by and large you should get similar results).
E.g., with about 97% probability, bit 7 should be 0; with  92% probability, bit 5 should be 1, etc.
So if our bit average is leaning the opposite way, then the corresponding bit of the key byte 
is probably 1. The imbalance is smallest for bits 0 and 2, which are therefore the hardest 
to determine in key bytes this way - e.g., with 30 ctxt bytes per key byte, bit 2 can be inferred
correctly with probability 2/3 or so (chance of majority vote being correct 
based on a binomial distribution). 
This is poorer than the scoring method because correlations between bits were ignored - but 
those can be included too. For example, for bits 2 and 3 the respective probabilities of 
joint 00, 01, 10, and 11 occurrences in English text are

```python
P00 = 0.372
P01 = 0.147
P10 = 0.271
P11 = 0.209
``` 

I.e., if we already determined that bit 3 is $$x$$, then the relative difference in the probabilities 
P0x and P1x gives bit 2 much more reliably than the small +-2% asymmetry in the single-bit probability
P2 we saw earlier.

#### Extracting XOR-red key bytes

The last ingredient we need is that, in principle at least, we could use bit correlations to determine 
the XOR of any two key bytes via 

$$c_i \hat{} c_j = (p_i \hat{} k_{i \mod N}) \hat{} (p_j \hat{} k_{j \mod N}) = (p_i \hat{} p_j) \hat{} (k_{i \mod N} \hat{} k_{j \mod N}) \ ,$$

since that also holds for a particular bit $$b$$ of each quantity: 

$$c_i(b) \hat{} c_j(b) = [p_i(b) \hat{} p_j(b)] \hat{} [k_{i \mod N}(b) \hat{} k_{j \mod N}(b)] \ .$$

So if the bit correlation 
between ctxt bytes is not what is expected from the ptxt (i.e., for English text),
then that bit in the corresponding two key bytes likely has opposite values. 
Key space then collapses to just 256 bytes
(one of the key bytes), relative to which we could determine all other bytes. 

There is a catch, however. Accuracy 
is much suppressed, so we need *a lot* of ctxt to get all bits. 
To see that, suppose that ptxt bytes were independent,
which is clearly just an approximation but one that should get better and better the larger $$|i - j|$$ is. Then we
are randomly drawing those bytes from the same frequency distribution. Let the 0-1 asymmetry for the given
ptxt bit be $$x$$, i.e., the chance for the bit to be 0 or 1

$$P_0 = 1/2 - x \ ,$$

$$P_1 = 1/2 + x \ .$$

The chance for the same bit to be 0 or 1 in $$p_i \hat{} p_j$$ is then

$$P_{00} + P_{11} = P_0^2 + P_1^2 = 1/2 + 2 x^2 $$

and

$$ P_{01} + P_{10} = 2 P_0 P_1 = 1/2 - 2 x^2 \ , $$ 

respectively, i.e., the asymmetry after XOR is only $${\cal O}(x^2)$$.
Nevertheless,
for very large asymmetries, e.g., bits 7 or 5 in English text, this approach is totally viable.
Notice, by the way, that the sign of $$x$$ drops out, so the only thing that matters is the
magnitude of the initial asymmetry, not its direction.


#### Details of bit determination

It is instructive to look into bit determination in full detail.
Suppose we know that on average bit $$b$$ in the ptxt is $$\bar p$$. We then measure
the same bit for XOR-red ctxt bytes at key positions $$i$$ and $$j$$, and find on average $$\bar c$$. 
Given $$\bar p$$ and $$\bar c$$, is bit $$b$$ in $$k_i \hat{} k_j$$ more likely to be 0 or 1?

The simplest approach may be to say that if an average bit is larger than 1/2, 
then the bit is more likely to be 1,
while if it smaller than 1/2, then the bit is more likely to be 0. Assuming that ptxt
bits at different byte positions are independent,
the probability for two ptxt bits to be the same - so their
XOR to be 0 - is 

$$P_{same} = \bar p^2 + (1 - \bar p)^2 \ .$$

So, in effect, we round
$$P_{same}$$ and $$\bar c$$ to 0 or 1, and deduce the keybit XOR value based on the table

```python
#  Psame   cbar    ki^kj
    0      0        0
    0      1        1
    1      0        1
    1      1        0
```

A more sophisticated *likelihood-based approach* gives just the same. 
Suppose that $$k_i(b) \hat{} k_j(b) = 0,$$ so the ptxt XOR average
should agree with the ctxt XOR average for bit $$b$$, and $$n$$ ctxt position
pairs correspond to the given key position pair $$(i,j)$$. Then, precisely $$n$$
terms went into the XOR average $$\bar c$$, some $$k$$ of which were 0, while $$(n-k)$$ of which
gave 1. The probability for the underlying ptxt bit XOR values to be 0 is $$P_{same}$$. *Assuming* that
these represented $$n$$ independent measurements, the probability for getting $$k$$ zeroes and $$(n-k)$$
ones, in whatever particular order, is

$$P_{keep} = P_{same}^k (1 - P_{same})^{n - k} \ .$$

If $$k_i(b) \hat{} k_j(b) = 1$$, however, then the ctxt XOR bit gets flipped in all $$n$$ terms,
so the same $$\bar c$$ outcome has probability

$$P_{flip} = (1 - P_{same})^k P_{same}^{n - k} \ .$$

We should infer $$k_i(b) \hat{} k_j(b)$$ by taking whichever of the two options, keep or flip, is more likely,
i.e., 

$$k_i(b) \hat{} k_j(b) = 0 \ \ {\rm if} \ P_{keep} > P_{flip} \ .$$

But

$$ \frac{P_{keep}}{P_{flip}} = \left(\frac{P_{same}}{1 - P_{same}}\right)^{2k - n} \ , $$

so we need to *flip* when $$P_{same} > 1/2$$ and $$k > n/2$$ (i.e., $$\bar c < 1/2$$),
or when $$P_{same} < 1/2$$ and $$k < n/2$$ (i.e., $$\bar c > 1/2$$). Precisely the
same conditions as in the table above.

**EDIT:** after typing this up, I realized that in the presence of *any* bit asymmetry, i.e.,
when $$\bar p \ne 1/2$$, we have $$P_{same} > 1/2$$, which always rounds to 1. So only the
bottom half of the table plays a role, which means that
the key bit XOR is always the opposite of the ctxt bit XOR.


### Breaking double XOR

Alright, on to breaking double XOR. The case when we have lots of ctxt is easy 
because we can consider double XOR equivalent to single-XOR with an encryption
key that is as long as the least common multiple of the two key lengths, $$N_1$$ and $$N_2$$.
The worst-case scenario is then a key length of $$N_1 N_2$$ 
when $$N_1$$ and $$N_2$$ are relative primes. 
With roughly $$30 N_1 N_2$$ ctxt bytes this is trivially breakable
using the single XOR solver - that was the **Baby double XOR** challenge at the CTF.
The real question is how to break double XOR given much less ctxt, 
ideally just about $$30 (N_1 + N_2)$$ bytes, but definitely less than $$N_1 N_2$$ bytes.

#### Key lengths

The first goal is to figure out the lengths of the two keys.
For definiteness, assume $$N_2 > N_1$$. 
The idea is the same as for single XOR: construct ctxt correlations that are independent of 
*both* keys. Take a ctxt position $$p$$. We can construct the set of all other positions $$p_2$$ that
are the same as $$p$$ modulo $$N_2$$. Let us call the set of such $$(p, p_2)$$ pairs

$$S_2 = \{ (p, p_2):  p = p_2\!\!\!\! \mod N_2, \   p \ne p_2 \} \ .$$ 

Then, for all pairs in $$S_2$$,  $$c_p \hat{} c_{p_2}$$ is independent of the second key. 
What about
the first key, though... Well, we can take each pair and compute their counterparts
modulo $$N_1$$:

$$(i, i_2) = ({p\!\!\!\! \mod N_1}, \ p_2\!\!\!\! \mod N_1) \ .$$

If $$i = i_2$$, then $$c_p \hat{} c_{p_2}$$ 
does not depend on the first key either 
but the number of such ($$p$$, $$p_2$$) pairs is small, so statistics will be poor. 
In fact, unless we get more than $${\rm lcm}(N_1, N_2)$$ ctxt bytes, there is not a single such pair.

However, we can use a much larger set if we
pair $$(p,p_2)$$ with all other pairs $$(q, q_2)$$ from $$S_2$$ that give the *same* 
($$i$$, $$i_2$$) mod $$N_1$$.
Then, the quadruple XOR

  $$c_p \hat{} c_{p_2} \hat{} c_q \hat{} c_{q_2} = p_p \hat{} p_{p_2} \hat{} p_{q} \hat{} p_{q_2}$$

is independent of both keys, so we can exploit bit asymmetries again.
The price looks steep though - after quad-XOR any modest bit asymmetry $$x$$ drops
to $${\cal O}(x^4)$$. E.g., with the independent ptxt bytes assumption,
the chance for getting a 0 result is

$$
P_{0000} + P_{0011} + P_{0101} + P_{0110} + P_{1001} + P_{1010} + P_{1100} + P_{1111} \\
= P_0^4 + 6 P_0^2 P_1^2 + P_1^4 = 1/2 + 8 x^4 \ .
$$

But for bit 7, this is still $${\cal O}(1)$$ because the original bit asymmetry is super large (for English text),
 so we get a jolly good discriminator. We simply try progressively larger $$N_1$$, $$N_2$$ pairs and
look for a drop in the average of bit 7 of the above quad-XOR. For the Double XOR challenge,
this yields $$\boldsymbol{N_1 = 53}$$, $$\boldsymbol{N_2 = 71}$$.


#### Obtaining keys

Next, deduce the keys, $$K^{(1)}$$ and $$K^{(2)}$$. 
The main strategy will be to *reconstruct $$K^{(1)}$$
with the bit correlation method discussed at the end of the single-XOR section,
then XOR the ctxt with it, and solve for $$K^{(2)}$$ using the single XOR solver*.
For any of our familiar $$(p,p_2)$$ pairs, we have

$$ c_p \hat{} c_{p_2} = p_p \hat{} p_{p_2} \hat{} K^{(1)}_i \hat{} K^{(1)}_{i_2} \ .$$

So averaging a given bit of $$c_p \hat{} c_{p_2}$$ for those pairs that give the same fixed $$(i,i_2)$$,
and comparing to expectation for English text, we get the corresponding bit of 
$$K^{(1)}_i \hat{} K^{(1)}_{i_2}$$. Math will be worth a few dozen words in what soon follows,
so let us write this **predictor for bit $$\boldsymbol{b}$$ of $$\boldsymbol{K^{(1)}_i \hat{} K^{(1)}_{i_2}}$$**,
in shorthand, as

$$O_{i,i_2}(b) = [ \langle c_p(b) \hat{} c_{p_2}(b) \rangle > 1/2 ] \hat{} [ \langle p_p(b) \hat{} p_{p_2}(b) \rangle > 1/2] \ .$$

This is the same as the bit predictor from the single-XOR section,
except using a restricted set of ctxt position pairs
(e.g., if the ctxt and ptxt bit-XOR averages are both above 1/2, so 
upon rounding we would estimate those to be 1, 
then $$O$$ gives $$1 \hat{} 1 = 0$$, i.e., that bit $$b$$ is the same 
at the respective positions $$i$$ and $$i_2$$ in $$K^{(1)}$$).
Now set $$i = 0$$ and let $$i_2 > 0$$ vary, cycle through all $$b$$, and we should get the full $$K^{(1)}$$, 
except with all its bytes XOR-red with the very first byte of key 1. But that itself is no problem. 
Double XOR encryption is invariant under XOR-ring both keys with the same constant value,
so we can simply consider the first byte of key 1 zero (in other words, XOR both keys with $$K^{(1)}_0$$).
Then the predictors $$O_{0,i_2}(b)$$ will give us $$K^{(1)}$$, after which we use the single-XOR solver to get 
$$K^{(2)}$$.

Great plan but the fly in the ointment is statistics - there is not enough
ctxt in the challenge 
to reliably determine *all* the needed ctxt bit asymmetries, so
we mispredict parts of $$K^{(1)}$$, and thus we do not get $$K^{(2)}$$ either. Luckily, we can
improve accuracy a lot. 
Notice that, in the method sketched above, information in $$O_{i,i_2}$$ with $$i>0$$ 
is totally ignored. We can, however, use those predictors to second-guess
the direct predictor $$O_{0,i_2}$$ via

$$\tilde O_{0, i_2}^{(i)}(b) = O_{0, i}(b) \hat{} O_{i,i_2}(b)$$

because for any $$i \ne 0, i_2$$

$$ K^{(1)}_0 \hat{} K^{(1)}_{i_2} = 
   (K^{(1)}_0 \hat{} K^{(1)}_{i}) \hat{} 
   (K^{(1)}_{i} \hat{} K^{(1)}_{i_2}) \ .$$

So we have, in fact, one direct and $$(N_1 - 2)$$ indirect ways to estimate a given bit
in the XOR of any two bytes in $$K^{(1)}$$.
Taking a simple majority vote among all the $$(N_1 - 1)$$ estimators ($$O$$ and all the $$\tilde O^{(i)}$$) 
improves accuracy greatly 
(you can probably come up with a
scheme that weighs the different pieces more appropriately).

If you generate some sample problems using the key lengths in the Double XOR challenge,
you find that this technique works pretty well for 
reconstructing bits 4, 5, 6, and 7 everywhere in $$K^{(1)}$$. In addition, bits 1 and 3 come out 
mostly right. However, bits 0 and 2 are basically a toss-up.


#### Adaptive key refinement via $$n$$-grams

The imperfectly reconstructed $$K^{(1)}$$, followed by the single-XOR solver for $$K^{(2)}$$,
typically gives final reconstructed ptxt consisting of mostly spaces and letters, 
and practically always printables, 
but it is all gibberish.
Except for character set and character frequencies, 
it does not much resemble English. The final step of the solver is to 
*adaptively refine the reconstructed $$K^{(1)}$$* (especially bits 0 and 2) 
*to make the final reconstructed ptxt more English-like*.
There are far too many bits to vary
for an exhaustive search, so we need some heuristics too.

**Scoring function.** The heart of this is an "English score" based on $$n$$-gram likelihoods. The same 
approach was used to solve the [2TP 
problem](https://github.com/b01lers/b01lers-ctf-2020/tree/master/crypto/300_2TP/sol) at b01lersCTF 2020. 
Taking a large body of English text, one can determine the conditional probability
	
$$P_n(p_i|p_{i-1},...,p_{i-n+1})$$

for a given character $$p_i$$ to be present at position $$i$$ in the text, given the preceding 
$$(n-1)$$ characters in the text. Making the assumption that each subsequent character is drawn from 
the same $$P_n$$ distribution (i.e., the text is a [Markov chain](https://en.wikipedia.org/wiki/Markov_chain)), 
we can estimate the probability of any text section as

$$P(p_i,p_{i+1}, .....,p_{i+k}) = \prod_{j=0}^k P_n(p_{i+j}|p_{i+j-1},...,p_{i+j-n+1}) \ .$$

This is just the scoring function we need. If we randomly change a couple characters in
the section, even if we flip all of those to high-frequency characters, 
we are likely to get improbable $$n$$-grams (gibberish), and the probability will drop.
There is a truncation at depth $$n$$, so
we want as large $$n$$ as possible. But then an exponentially
large body of sample text is required to infer $$P_n$$ reliably.
The end result is that we take whatever $$n$$ we can afford ($$n = 4$$ is quite good in practice).

There are some subtleties, of course. First, to prevent super small numbers, it is more convenient to 
follow the log of the probability (log-likelihood):

$$\ln P(p_i,p_{i+1}, .....,p_{i+k}) = \sum_{j=0}^k \ln P_n(p_{i+j}|p_{i+j-1},...,p_{i+j-n+1}) \ .$$

So we get negative numbers, and the goal will be to find the least negative score.
Second, near the beginning of the text we do not have $$(n-1)$$ preceding characters, so
we will also need to build $$(n-1)$$-gram, $$(n-2)$$-gram, etc, down to 1-gram probabilities. E.g.,

$$P(p_0, p_1, ..., p_k) = P_1(p_0) P_2(p_1|p_0) P_3(p_2|p_1 p_0) P_4(p_3|p_2 p_1 p_0) \dots $$

Finally, any finite body of text will have holes because low-probability $$n$$-grams may end up missing
completely,
and low counts will be statistically unreliable as well. This can be remedied by a 
clever [Witten-Bell smoothing](https://www.ee.columbia.edu/~stanchen/e6884/labs/lab3/x207.html) 
procedure that gradually falls back onto lower $$n$$-grams in the absence of high $$n$$-gram data
(the 2TP solver had this too).


**Search heuristics and speedups.** All we need now is a strategy for adjusting bits 0-3 in all $$K^{(1)}$$ bytes. 
That is a lot, 16 possibilities per key byte ($$4N_1$$ bits). 
It is natural to try optimizing short contiguous 
sections at a time. The most extreme simplification would be to sweep through the key and optimize 
just one byte at a time (maximize the score for the reconstructed ptxt) 
but that turns out to be rather ineffective in practice. On the other hand, 
the next simplest 
shortcut, optimizing two consecutive bytes at a time - positions 0 and 1, then 1 and 2, then 2 and 3, 
..., $$N_1 - 2$$ and $$N_1 - 1$$, and finally $$N_1 - 1$$ and $$0$$ again, works amazingly well
(it is important to wrap around at the end, or else we introduce a subtle bias and may get stuck on 
an imperfect solution where all bytes of $$K^{(1)}$$ except the first or last one are correct). 
At the end of the sweep, XOR all $$K^{(1)}$$ bytes with $$K^{(1)}_0$$.
One full sweep
then takes $$16^2 N_1 = 256 N_1$$ tries, totally within reach given the typical constraints of a 2-day CTF,
albeit a tad intensive computationally.

We can do far better by starting out with *restricted sweeps*. First try to fix only one bit at a time 
(i.e., XOR key 1 bytes with 0, 1, 2, 4, 8), which takes only $$5^2 N_1$$ tries per sweep, so it is 
quicker by one order of magnitude. Obviously this does not always get the right result but it 
does take care of most bit errors quickly. Once single-bit sweeps converge, expand search space to flipping two 
bits, three bits, and only as the last step do full 4-bit sweeps.

Near the end of the single-bit sweep phase, 
you already start seeing word snippets in the reconstructed 
ptxt, including some unique-looking segments. Exploiting these as constraints 
speeds up the remainder of the search drastically because 
we can *consider $$K^{(1)}$$ bytes that are involved in these fragments frozen*.
(It is convenient to read such phrases in from an external file at the beginning of each sweep, 
and then you can add more and more words or phrases to the file while the search is running.)
Final advice,
if you code this in python: do not forget
the [numba](http://numba.pydata.org/) package - its JIT acceleration
is a *massive* speedup. With numba, 
the full Double XOR challenge
can be solved in 5-6 minutes without utilizing any phrase constraints
(and finding the lengths of the keys takes just a few seconds).


### Closing thoughts

So, obviously, double XOR encryption is breakable. Based on the above technique, 
it should be possible to make a rigourous estimate
for just how much ctxt - and CPU ;) - you need to succeed with high probability.
A related thought is that once you employ natural language constraints, 
regular XOR encryption can probably be decrypted using 
many fewer than $$30N$$ ctxt characters...

[back](/)

