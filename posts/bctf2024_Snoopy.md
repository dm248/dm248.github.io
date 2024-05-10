---
usemathjax: true
---
# Snoopy (@ [b01lers CTF 2024](https://ctftime.org/event/2250))

This was a [crypto chal](https://github.com/b01lers/b01lers-ctf-2024-public/tree/main/crypto/snoopy) written in python 
where you had to leverage side-channel info. The underlying math puzzle was very similar to the subset-sum problem 
(e.g., knapsack crypto), so it was constructed to be breakable using lattice techniques (LLL). For flavor, the code was 
infused with some computer and microelectronics related mumbo-jumbo terminology.

Here is the [solver code](https://github.com/b01lers/b01lers-ctf-2024-public/tree/main/crypto/snoopy/solve) from the 
public bctf2024 repo.

### Setup and overview

There was a $$65 \times 65$$ rectangular grid ("die"), the points along the four sides formed a 32-byte prepared state (256 "pins" that 
were either zero or a "high" value of 3.3). An algorithm was then applied that propagated the state to the inside $$63\times 63 = 3969$$ 
points. You had no access to the edge points but, rather, had to reconstruct those 32 bytes from knowledge of the values along a rectangle 
inside the board. There was some freedom in where you put the rectangle but it could not be too large, could not be too narrow, and could 
not be too close to the edges of the board.

The state had some fixed bytes (initialized on "boot"), a predictable counter byte, 16 bytes that came from AES encryption of the flag 
(with`bctf{`and`}`stripped off), and 6 bytes of the originally 8 random bytes that were used to generate the key (these were concatenated 
four times for a 256-bit key). So practically 22 bytes (176 bits) of the state were random, which you needed to infer somehow. The 
algorithm did not propagate inside the values at the 4 corners of the board at all (lowest bit of bytes 0, 8, 16, and 24), so **after 
reconstructing the remaining 172 bits on the edge, you needed to bruteforce $$ \mathbf{2^{4 + 16} = 2^{20}}$$ possibilities for the corner 
values and the missing key bytes**.

```
GRID:    top left corner is (0,0), bottom right corner is (64,64)
         -> so rows span 0-64, columns span 0-64

STATE:   32 bytes along edges (rows 0 & 64, columns 0 & 64)

Byte arrangement:  each byte = 8 pins, pin index increases clockwise

    0 1 2 3 4 5 6 7
    - - - - - - - -
 31|               | 8    AES ctxt:  all (16) even bytes
 30|               | 9
 29|               |10    AES key:   bytes 19, 21, 23, 25, 27, 29
 28|  This is      |11               (also bytes 31 & 0, but get overwritten)
 27|  the "die"    |12
 26|               |13    counter:   byte 31
 25|               |14
 24|               |15    fixed:     all other bytes
    - - - - - - - -
    2 2 2 2 1 1 1 1
    3 2 1 0 9 8 7 6
```

For each encryption you got a new random key, thus a new state, which was then used to set the inside points again - so there was no value 
in requesting multiple encryptions during one session, nor in collecting multiple encryptions over several sessions. For fun, I 
coded some logic that ended the session after 256 encryptions.


### Propagation algorithm

The algorithm first randomized the $$63\times 63$$ inside points, and then repeatedly replaced that inside square with a new one in which 
each point had the average of the preceding values of its four nearest neighbors. (Notice that for edges of this inside square, i.e., 
rows 1 & 63 and columns 1 & 63 of the full "die", these neighbors include the "pins" on the perimeter of the full die). Replacement went 
on until the maximum per-site change along the square was less than $$\epsilon = 10^{-14}$$, or until 200,000 iterations were reached in 
which case the chal bailed out with an error.

Some teams have probably recognized that this is how one might solve the [Laplace 
equation](https://en.wikipedia.org/wiki/Laplace%27s_equation) in electrostatics on a 2D grid - the input "pins" are voltages along the 
edge of the grid and the values inside are the electric potential at each site. Some nice-to-have but otherwise inessential background 
knowledge... The only thing that really mattered was that only nearest neighbors were involved.

There was a small refinement (`turbo`parameter) - in each sweep we altered values by 1% less than what the new average would have 
dictated. This was a genuine speedup that reduced iteration count by about a third. Normally you would accelerate convergence by 
*amplifying* changes (called [successive over-relaxation](https://en.wikipedia.org/wiki/Successive_over-relaxation)), i.e., use`turbo < 
0`, however, the particular algorithm here is known to become unstable in that case (the 
[variant](https://en.wikipedia.org/wiki/Gauss%E2%80%93Seidel_method) that simply overwrites the original grid values as it goes would work 
fine that way but I thought it would be harder to reason about).


### Attack strategy

The key 
observations are that

* 1) $$\epsilon$$ is practically zero

* 2) so, at termination, each point is the average of its neighbors; i.e., **we end up with the solution to some 
large linear system** (apart from very small errors)

* 3) **the system is homogeneous** - if all outside points are zero, then the inside points are all zero as well

* 4) the solution is independent of the value of the 4 corner points, so for those one must try all $$2^4$$ possibilities

    
In other words, we just converge to the solution of an equation $$A x = B c$$, where $$x$$ is a 3969-component vector representing the inside 
values, $$c$$ is a 256-component vector of the outside points, $$A$$ is some fixed $$3969 \times 3969$$ matrix, and $$B$$ is another fixed 
$$3969\times 256$$ matrix. The solution can be constructed by solving

$$A x^k = B v^k \qquad(k=0, \dots, 255) \ ,$$

where $$v^{k}$$ is a vector of all zeroes except for a 
single value of 1 at component $$k$$, and then combining those results according to the bits (0s and 1s) in $$c$$:

$$c = \sum_k c_k v^k  \qquad \Rightarrow \qquad x = \sum_k c_k x^k \ .$$

Here, $$x^{k}$$ is the response to a "high" pin at $$k$$ (the analogous entity would be the Green's function in physics), which we can 
compute by simply running the algorithm with input $$c = v^{k}$$.

Of course, one did not get all the inside points $$x$$ but only a subset (projection) $$\tilde x$$ consisting of some $$K \le 80$$ points 
on the chosen rectangular "snoop" loop. Nevertheless, that subset satisfies an analogous relation

$$\tilde x = \sum_k c_k {\tilde x}^k \ . $$

So in the end, this just boils down to $$K$$ simultaneous subset sum problems - one problem for each component of $$\tilde x$$, where each 
problem is solved by the same $$c$$. We could call this the **"functional subset sum problem"** - you are given a bunch of template 
functions plus a function that is some linear combination of those with coefficients strictly drawn from $$\{0,1\}$$, and your job is to 
figure out the coefficients used. Equivalently, you need to find the closest function to your input that you can produce with coefficients 
from $$\{0,1\}$$.


### Snoop loop placement

Alright, and where to place the loop that gave side-channel info? 

First, realize that knowing values along the outline of a rectangle is equivalent to knowing all values inside as well because you can get 
those by solving the linear system (remember, only nearest neighbors are involved). In other words, **inside points carry no additional 
information**, only the perimeter of the rectangle matters. So you want **a rectangle as large as possible** to get maximal info 
(e.g., any rectangle gives more information than another one that is fully inside it). The maximum allowed circumference was 80 points. 
Purposefully fewer than the 176 unknown bits of the input state but that is fine - the 80 points have values from a continuous range, 
i.e., each one can hold several bits of information.

What about position and shape? Well, there was a certain symmetry to problem - the 16-byte *AES-encrypted block* was scattered quite evenly 
along all four sides of the grid, which naturally suggests a square loop centered at the middle. At maximum circumference, this means 
**upper left corner (22, 22), lower right corner (42, 42)**. If you plot the response functions, you can see that they generally drop with 
distance away from the "high" pin. So, to be equally sensitive to all input bits, you need a loop with points that are about same distance 
away from each pin. That said, the AES *keys* were mostly on the left side of the grid, so I felt that there might be some room for a modest 
*positioning* bias/optimization. Which is why I ultimately left the choice up to you.

Actually, the shape of the rectangle is pretty unimportant. As discussed above, knowing values on a rectangle implies knowing all the 
points inside too. Fine. It is also trivial to infer outside values in triangular regions sitting on the sides (you go layer by layer 
outwards, always filling in the value for the only unknown neighbor of sites):
```
       ++
      ++++
     ++++++
    XXXXXXXX
   +X******X+         X:  snoop loop  (7x8 => 26 points)
  ++X******X++           
 +++X******X+++       *:  inside points (5x6 => 30 points)
  ++X******X++ 
   +X******X+         +:  outside triangles
    XXXXXXXX            
     ++++++
      ++++
       ++

Inferring outside triangles:

    X******X
    XXXXXXXX     <-- consider points in this row (except leftmost and rightmost ones)
     ??????      <-- these are the only unknown neighbors, one for each site

```

*These outside points give no new information either*, of course. I did not want people to try getting to the original inputs in such a 
direct way, however, so there was a requirement to have a minimum gap of 10 points between the edges of the grid and the snoop rectangle. 
More importantly, thinking about the problem this way tells you that **snoop loops that have the same set of inferable points 
(circumference + inside + outside) provide identical information**. So instead of the 7x8 loop in the figure below, one could have chosen, 
equivalently, a 3x12, 5x10, 9x6, 11x4 or 13x2 loop. These loops all have the same circumference.


### Breaking the subset sums

Upon boot, the 32-byte initial state is`b"SSnnOOooPPyy@@BBCCttff22002244#\x00"`. After encryption, the state becomes 
`b".S.n.O.o.P.y.@.B.C.............?"`, where "." denotes unknown bytes, while "?" stands for a counter byte that holds the number of calls 
made to the key generation function (e.g., for the first flag encryption after boot, it is b"\x01", assuming you never picked the key 
generation option in the menu). Suppose the response to the 10 known bytes, i.e., 80 known bits of the state after encryption is $$x_0$$, 
and we subtract this off by taking $$\Delta x = x - x_0$$. We then have $$N = 176 - 4 = 172$$ unknown input bits to find via breaking the 
joint subset sum problem for $$\Delta \tilde x$$ (as discussed earlier, the response to state bits 0, 64, 128, and 192, corresponding to 
the four corners is zero, so these bits do not contribute to the sum and will have to be bruteforced).

**Starting point:** low-density subset sum solver via the [LLL 
algorithm](https://en.wikipedia.org/wiki/Lenstra%E2%80%93Lenstra%E2%80%93Lov%C3%A1sz_lattice_basis_reduction_algorithm), see Chapter 
3.10.2 of *Handbook of Applied Cryptography* by Menezes, Oorschot, & Vanstone. I prefer multiplying their lattice by 2 so that all values 
are integers.

For simplicity of notation, remap indices and label pins corresponding to the unknown bits to $$k = 1,$$ $$\dots,$$ $$N.$$ 
Responses to these along the side-channel loop are given by $$\tilde x^k$$. For a loop of $$K$$ points, each $$\tilde x^k$$ has $$K$$ 
components and we have $$K$$ simultaneous subset sum problems, so we consider an $$(N + K)$$-dimensional lattice with the $$(N + 1)$$ 
lattice vectors below:

$$M = \left( 
  \matrix{ 
 2 &  0  & 0 & \dots & 0  &  (\tilde x^1)_1 & (\tilde x^1)_2 & \dots & (\tilde x^1)_K \cr
 0 &  2  & 0 & \dots & 0  &  (\tilde x^2)_1 & (\tilde x^2)_2 & \dots & (\tilde x^2)_K \cr
 0 &  0  & 2 & \dots & 0  &  (\tilde x^3)_1 & (\tilde x^3)_2 & \dots & (\tilde x^3)_K \cr
 \cdots \cr
 0 &  0  & 0 & \dots & 2  &  (\tilde x^N)_1 & (\tilde x^N)_2 & \dots & (\tilde x^N)_K \cr
 1 &  1  & 1 & \dots & 1  &  \Delta \tilde x_1   & \Delta \tilde x_2   & \dots & \Delta \tilde x_K \cr
} 
\right)
$$

The left $$N$$ columns track coefficients, while the right $$K$$ columns track the sums. The linear combination that perfectly solves our 
problem would have its first $$N$$ components equal to $$\pm 1$$, while the last $$K$$ components equal to zero. This vector has an 
$$L_2$$ norm of $$\sqrt{N}$$, which may not be shortest, and may not even be among the vectors returned by LLL. Yet, just like in the 
low-density subset sum solver, if one scales the left or right sides of $$M$$ just right relative to each other, LLL returns the solution 
with high probability, often even outside the regime where the method is rigorously proven to work (and when LLL fails, you might still 
get lucky if you reconnect to generate a new problem).

In our case, proper scaling is not that trivial because for the correct solution the sums will *not* be exactly zero:

* we only solved the linear system up to some precision, starting from randomized inner points, so the solutions themselves have (small) 
random errors. Random fluctuations could be minimized by rounding values to, say, 9 significant digits.

* response vector components are doubles, while LLL needs integers (or rationals but we scaled up $$2\times$$ to have integers only). When 
we **scale and quantize** these to get integers, i.e., multiply by a factor $$Q$$ (say, $$Q = 10^9$$) and round, we make an error in each 
component. In sums, these errors get amplified - if rounding misses by $$\pm 0.5$$ per component, then after summing $$N$$ terms errors 
can reach $$\pm N/2$$.

So we need to balance how much accuracy we demand on the sums vs how small we want coefficients to be. Scaling the coefficient part of 
$$M$$ up by integer factor $$S_1 > 1$$ emphasizes smallness of the coefficients, while scaling the sum half of the matrix by $$S_2 > 1$$ 
prioritizes smallness of the sums. In the end what works (for $$Q = 10^9$$) is setting $$S_1 \approx 100$$, $$S_2 = 1$$, 
so we LLL-reduce the system

$$M' = \left( 
  \matrix{ 
 2S_1 &  0  & 0 & \dots & 0  &  (\tilde y^1)_1 & (\tilde y^1)_2 & \dots & (\tilde y^1)_K \cr
 0 &  2S_1  & 0 & \dots & 0  &  (\tilde y^2)_1 & (\tilde y^2)_2 & \dots & (\tilde y^2)_K \cr
 0 &  0  & 2S_1 & \dots & 0  &  (\tilde y^3)_1 & (\tilde y^3)_2 & \dots & (\tilde y^3)_K \cr
 \cdots \cr
 0 &  0  & 0 & \dots & 2S_1  &  (\tilde y^N)_1 & (\tilde y^N)_2 & \dots & (\tilde y^N)_K \cr
 S_1 &  S_1  & S_1 & \dots & S_1  &  \Delta \tilde y_1   & \Delta \tilde y_2   & \dots & \Delta \tilde y_K \cr
} 
\right)
$$

where $$\left\{ \tilde y^k\right\}$$ are scaled and quantized response vectors, while $$\Delta \tilde y$$ is the scaled and quantized 
measurement vector along the side-channel loop (with $$\tilde x_0$$ subtracted before scaling). If in the reduced basis we have a vector 
with its first $$N$$ components all $$\pm 1$$, then we found the solution. 
Either $$+1$$ stands for a 1 bit, and $$-1$$ for a 0 bit, everywhere, or vice 
versa.

### Final bruteforce

The 172 bits extracted via LLL give us the AES encrypted flag, apart from 4 missing bits, and 6 of the 8 random bytes used for the AES 
key. Encryption mode was ECB, so we know that there are exactly 16 bytes inside the flag (and there is no padding), or else the AES 
encryption routine would have complained. This is too long for attacks on the flag itself, so just generate all 16 possible ciphertexts, 
decrypt those with all the 65536 possible key completions, and check whether the result is printable (has ASCII values 0x20 - 0x7f). The 
calculation takes roughly a minute in python. 

There is a modest probability for getting a viable candidate by pure chance, about 

$$P = 1 - [1 - (96 / 256)^{16}]^{2^{20}} \approx 15\%$$ 

if we assume each decrypted byte is uniformly random and independent. When that happens, one can 
just apply common sense to select the right flag or redo the challenge to see which flag candidate is stable.

### Some design considerations

The idea of doing subset sum with vectors came fairly early but it took some tweaking to get it to work. Originally the 
plan was to just put the 16-byte ctxt block plus a 16-byte key on the perimeter (total 256 bits), however, LLL always 
missed the correct solution. It was the usual culprit - "fake" short vectors in the lattice that did not use the target 
vector (last row of the input lattice) at all. Specifically, the worst was that responses to a high input at opposite 
sides right near a corner of the grid (e.g., at (0,1) vs (1,0)) are almost identical, so LLL can just happily take 
their difference to construct rather small vectors.

I almost settled on having only 128 unknown bits away from corners, which would have been a problem with a few rounds 
of "here is the key, snoop the ctxt for some random ptxt, and give me the ptxt" but that felt less ballsy than giving 
people direct encryptions of the actual flag. A variant of that idea was to spread the 16-byte ctxt along all four 
edges of the grid in a way that near each corner we only use bits on one side of the corner - a simple way to do this 
was having "memory" locations increment by the equivalent of 2 bytes at a time (the in-chal story for this was 
"historical limitations" just like, e.g., for a QWERTY keyboard). Eventually I realized, however, that LLL does work 
fine even when I put unknown bits along both sides near - up to two - corners. Which then led to the final arrangement 
in the problem.

The $$O(10^6)$$ bruteforce was a standard play-it-safe ingredient - you normally want a solve to take at least a 
minute to prevent people from constantly rerolling for a luckier problem. The key only had 64 bits of entropy, 
a super-determined team could make a million connections during a two-day competition, and who knows what computing resources 
people might have at their disposal.. :)

---

[back](/)
