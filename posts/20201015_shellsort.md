---
usemathjax: true
---
# Curious case of (13,4,1) shellsort

##### Oct 15, 2020

I tend to think back fondly of the good old times when I was a student but I must have had it easy.
Here is an assignment someone else got recently: *Find the worst-case input for (13,4,1) shellsort
on 25 elements*. The official [solver](https://github.com/dm248/shellsort-13-4-1/blob/main/naive_solver.java) offered was simple, just shy of one *billion* core years to run
(trying all 25! permutations). So that got me curious, is the problem doable?
Well, it turns out that it is, though not quite that way (check the [code](https://github.com/dm248/shellsort-13-4-1) on GitHub, if interested).

First off, $$25!$$ is ballpark $$10^{25}$$, clearly hopeless - one needs to bring the work down to a trillion 
($$10^{12}$$) or so cases to be in business. Assume that all elements are distinct, and let those be 0...24
in sorted order (values are irrelevant, only $$<$$ $$>$$ relationships matter in sorting anyway). Recall what 
(13,4,1) shellsort does:
it uses insertion sort to [*h*-sort](https://en.wikipedia.org/wiki/Shellsort) the elements 
first using $$h=13$$, then by $$h=4$$, and finally $$h=1$$.
Here, *h*-sorting means separately sorting every subset of elements that are at the same position modulo $$h$$ 
(there are $$h$$ such subsets given by the position 0...($$h$$-1) of the first element in the set),
and the result is said to be *h-ordered*.
In what follows, it will be useful to define *unsorting* as the operation that takes a sorted set $$S$$
and gives the sets that would sort to $$S$$ (this is basically taking all permutations of $$S$$, including
the one where $$S$$ is kept unchanged).

### Strategy

Suppose the 13-sort does $$x$$ swaps, the 4-sort $$y$$, and the final 1-sort $$z$$ swaps. Then, in total, 
there are $$x + y + z$$ swaps, and we want to maximize this sum. The first step, 13-sort, compares 12 
pairs of elements - at positions (0,13), (1,14), ... (11,24) - so there are $$2^{12}$$ ways to unsort this. We can 
always pick the one that takes the maximal swaps to 13-sort, i.e., $$\boldsymbol{x = 12}$$. Considering the results of 
the 13-sort, there are still $$25! / 2^{12}$$ possibilities, far too many to enumerate. Then comes the 
4-sort that operates on groups of 7, 6, 6, and 6 elements (positions $$4 k + a$$, where $$k$$ is integer, 
while $$a = 0,1,2,3$$). The number of possible results of a 4-sort is simply the number of ways one can 
split 25 elements into 7 + 6 + 6 + 6, which is $$25! / (7! \cdot  6!^3)$$ ~ $$\bf 8 \cdot 10^{12}$$. Even fewer, if we impose 
13-ordering as well (13 and 4 are relative primes, so a 4-sort maintains 13-ordering, it does not spoil 
the prior 13-sort - see, e.g., *Theorem K* in Section 5.2.1 of Knuth's big book). 
Thus, this does look doable, *if* we can efficiently iterate through all 4-ordered arrangements - 
more on that later.

By generating 4-ordered elements, we can count how many swaps the final 1-sort would take on those, 
i.e., compute $$z$$. The only question left is how large $$y$$ can be. Surely, $$\boldsymbol{y \le 66}$$ (the worst case 
is 1+2+3+4+5 = 15 for an [insertion sort](https://en.wikipedia.org/wiki/Insertion_sort) 
on 6 elements, and 1+2+3+4+5+6+7 = 2 on 7 elements $$\Rightarrow$$ 3\*15+21 
= 66 for $$y$$ in the 4-sort we have) but we must impose 13-ordering as well, 
and we are maximizing the sum $$y + z$$, so we need to find the highest
$$y$$ reachable from each individual 4-ordered result. 
4-unsorting generates $$7! \cdot 6!^3$$ options for each arrangement, so at first sight we are back 
to $$25!/2^{12}$$ 
possibilities and gained nothing. However, we do not need to check the unsorts for every single 
4-ordered result - we only(!) need those that yield many enough swaps to be viable contenders for the 
worst arrangement possible. So, if we know a permutation of 0...24 that gives quite high swap count in 
(13,4,1) shellsort, we only need to beat that one - and this radically truncates the search space. 
E.g., suppose we need to beat 100 swaps, and we have $$z = 10$$. Then, we can skip that arrangement 
completely because even with maximal $$x =12$$, $$y=66$$ there are only 12+66+10 = 88 swaps. It gets even better.
Suppose $$z$$ = 30. Now 12 + 66 + 30 > 100, so this configuration must be investigated further.
Still, when we do the 4-unsort, we only need to check possibilities that yield a $$y$$ above 
100 - 30 - 12 = 58, and that is a small fraction of the full search space of all possible 4-unsorts.

**The general strategy is then this:**

* 0) find an initial candidate for (13,4,1) shellsort that yields a particularly high swap count upon sorting
* 1) generate all (13,4)-ordered arrangements involving permutations of 0...24
* 2) for those permutations that take many enough swaps to 1-sort,
     generate all 4-unsorts that are also 13-ordered **and** their swap count 
     in the 4-sort is enough to beat the candidate in 0)
* 3) if the candidate is beaten, print the arrangement that beat it (i.e., we just filter, and will sort 
the printout later)

Also, **code in C++** (or something similarly fast), and give the compiler as much information
as you can (templates come handy).


### Step 0: find said candidate

Finding candidates that are particularly bad
for (13,4,1) shellsort is straightforward, simply whip up a dumbed-down genetic algorithm that
has no pool, just the current best candidate:

* a) start from the candidate 0,1,...24
* b) mutate it by interchanging up to $$N$$ pairs of elements - after each swap 13-unsort to maximal $$x = 12$$, 
     and compute the number of swaps needed to (13,4,1)-sort the elements
* c1) if a mutation beats our candidate, replace it with the mutation
* c2) if we failed to improve, then generate a random candidate, mutate that one as in b), and 
replace the current candidate if a mutation beats it
* d) goto b)

(as someone told me this is basically a
[hill climbing](https://en.wikipedia.org/wiki/Hill_climbing) algorithm). Despite the random excursion
in c2), after a few thousand iterations the algorithm usually gets stuck. 
So set a finite limit on the iteration count,
and rerun. With $$N = 10$$ and 100,000 iterations, within a minute or so you get

* **[18,24,5,11,17,23,4,10,16,22,3,9,12,15,21,2,8,14,20,1,7,13,19,0,6]**

which takes $$(x,y,z) = (12,57,93)$$ swaps, **162 in total**.

This means that we need to find 4-ordered arrangements with $$z > 162 - 66 - 12 = 84$$, i.e., $$\boldsymbol{z \ge 85}$$,
and $$y + z > 150$$. As a crosscheck, we will search for $$\boldsymbol{y + z \ge 150}$$, 
so that the candidate we want to beat also gets printed by the algorithm.


### Step 1: generate all (13,4)-ordered configurations

As was discussed above, a 4-ordered configuration corresponds to a grouping of
25 numbers into sets of 7 + 6 + 6 + 6. It will be faster to generate this as 6 + 6 + 6 + 7.
So first pick 6 elements out of 25, which can be done in $$C(25,6) = 25!/(19! \cdot 6!) = 177100$$ ways, 
then pick another 6 from the remaining 19 (there are $$C(19,6) = 19!/(13! \cdot 6!) = 27132$$ ways),
finally pick another 6 from the remaining 13 (for this there are $$C(13,6) = 1716$$ options). The remaining
7 elements are those that were not picked yet. 

#### Counters for combinations

Each of the $$C(n,k)$$ combinations can be represented as a special $$k$$-digit counter, namely,
an ordered list of $$k$$ indices corresponding to the locations to pick from the list of still available 
numbers.
To step to the next combination, increment the last index in the list, if that overflows ( $$\ge n$$), 
then increment the index before that, if that also overflows ( $$\ge n-1$$), then increment the one before that, 
etc. Once an increment succeeds, set indices after that position to +1, +2 ... higher than
what the updated index was. E.g., in $$C(25,6)$$

* 0,1,**17**,22,23,24

will increment to

* 0, 1, **18**, 19, 20, 21

where the successfully incremented index is in bold. If the first index overflows ( $$> n - k$$),
then all combinations have been exhausted, so reset indices to 0...($$k$$-1).

Armed with a nested triple loop over the 3 counters (Python-style pseudocode below)

```python
for ctr1 in C(25,6):    # elements at locations 4k+1
   for ctr2 in C(19,6):  # 4k+2
      for ctr3 in C(13,6): # 4k+3
         set25 = reconstruct_permutation(ctr1, ctr2, ctr3)
         if is_13_ordered(set25):
            z = count_swaps(set25)
            if z >= zmin:
               ... find highest y ...
               ... etc ...
```

we can reconstruct in the innermost loop
each 4-sorted permutation of 0...24 from the actual counter states. The most naive solution
is to maintain an array indicating available positions, and repeatedly 
go through the array to find, for a counter index $$k$$, the $$k$$-th empty position, and use the number there.
Once done with a group, mark the locations picked for the group used, and move to the next one.
It looks like that this takes multiple sweeps through
the array, for each index in each group of the 6-6-6-7 split.
With a little thought, however, one can do it in $$O(25)$$ per group (no need to always start from the beginning of
the array, e.g., if subsequent counter indices are 16 and 18, then location 18 is just the second available 
location after where 16 was). 

By maintaining separate lists of available numbers before
each loop, in each loop you only need 1 pass (2 passes in the innermost one), and you shave off
about ~5% from the execution time. Helps a bit but with all these tricks the approach 
still would take roughly *100 core-days* to finish.


#### Innermost loop speedup

As usual in optimization, the primary goal is to speed up the innermost loop. There is a lot of 
repetition here, we practically do the same logic for each value of counter 3: reconstruction of a 
certain permutation of the last 13 unassigned elements and using that to complete the remaining group of 6+7 
in the 4-ordered permutation at hand. 
You gain nearly a *factor 10* in speed by precomputing and storing 
these permutations. The loops now become:

```python
for ctr1 in C(25,6):    # elements at locations 4k+1
   avail19 = find_available(ctr1)
   reconstruct_4k1(set25, ctr1)    # sets elements at 4k+1 in set25
   for ctr2 in C(19,6):  # 4k+2
      avail13 = find_available(ctr2, avail19)
      reconstruct_4k2(set25, ctr2, avail19)   # sets elements at 4k+2 in set25
      for i in range(1716):
         perm13 = permutation13_table[i] 
         reconstruct_rest(set25, perm13, avail13)   # sets elements at 4k+3 and 4k in set25
         if is_13_ordered(set25):
            z = count_swaps(set25)
            if z >= zmin:
               ... find highest y ...
               ... etc ...
```
where reconstruct_rest in the inner loop is now as simple (and fast!) in C++ as
```cpp
            for (int i = 0; i < 6; i ++) {
               int pos = perm13[i];
               set25[4 * i + 3] = avail13[pos];
            }
            for (int i = 0; i < 7; i ++) {
               int pos = perm13[i + 6];
               set25[4 * i] = avail13[pos];
            }
```

This cuts the running time to about 10-12 core-days, doable in **1-1.5 days on an 8-core machine**.


#### 13-ordering

We need (13,4)-ordered permutations, so we must impose 13-ordering as well. This means
satisfying 12 conditionals
that are always between two different sets:

* i) between sets (4k+1) and (4k+2):  [1] < [14], [5] < [18], [9] < [22]
* ii) between sets (4k+2) and (4k+3):  [2] < [15], [6] < [19], [10] < [23]
* iii) between sets (4k+3) and (4k):  [3] < [16], [7] < [20], [11] < [24], and
* iv) between sets (4k) and (4k+1): [0] < [13], [4] < [17], [8] < [21]

where [j] denotes the element at position j. Unlike in the pseudocodes above, grab every opportunity for early bailout:
i) can be checked before loop 3 (so the entire innermost loop can be skipped when a check fails), 
ii) before unpacking the group of 7 (so unpacking the last group can be avoided if any of the checks fail), 
but iii) and iv) must wait until the full permutation of 25 elements has been reconstructed.



### Step 2: undo the 4-sort

Now that we have a way to fish out each viable (13,4)-sorted sequence with $$\boldsymbol{z \ge 85}$$, we need
to check all different ways to undo their 4-sort. Before I get to that, just how many candidates
do we have? 

If you invest the time (~34 hours), you can collect the frequency distribution of $$z$$:

```python
# z   N(z)    N(>=z)
...
85 50236552 159482062
86 35338405 109245510
87 24568294 73907105
88 16869655 49338811
89 11430945 32469156
90 7635883 21038211
91 5022262 13402328
92 3247422 8380066
93 2060583 5132644
94 1280242 3072061
95 776784 1791819
96 458797 1015035
97 262787 556238
98 145284 293451
99 77095 148167
100 38986 71072
101 18621 32086
102 8298 13465
103 3396 5167
104 1248 1771
105 397 523
106 104 126
107 20 22
108 2 2
```

(the rightmost column gives the cumulative distribution, i.e., the number of permutations
with swap count $$\ge z$$). So among all (13,4)-ordered permutations, the highest $$z$$ value 
possible is 108, and in the $$z \ge 85$$ range of interest to us, there are about *160 million*
arrangements. I.e., we already avoided most of the work as this is only a small fraction
of the $$O(10^{12})$$ possibilities.

#### Quadruple loop

To undo the 4-sort, we need to take all permutations of elements within each of the 6-6-6-7 subgroups, 
of which there are $$6!^3 \cdot 7!$$ ~ $$\bf 2 \cdot 10^{12}$$.
A trillion in itself would be manageable but we need to do 
this for some 100 million candidates... 

There are a couple trivial speedups. First, the number of 
swaps in the 4-sort is just the sum of the swaps made while 4-sorting each of the groups, and the 
number of swaps is directly controled by the permutation used to undo the sort in the group. So the 
swap counts can be precomputed for each permutation, and you then only need to add those (much faster 
than tracking swaps in an actual 4-sort):
```python
for p1 in permutations(6):
   for p2 in permutations(6):
      for p3 in permutations(6):
         for p4 in permutations(7):
            cand2 = unsort(candidate, p1, p2, p3, p4)
            if is_13ordered(cand2):
               y = p1.swaps + p2.swaps + p3.swaps + p4.swaps
               if y + z >= 150:
                  print(cand2)
```
Second, the search can be pruned by checking conditions i) through iv) for 13-ordering
as early as possible (e.g., check i) before the loop with *p3*, ii) before the loop with *p4*).

But, by  far, the biggest improvement comes from bailing out whenever the maximum reachable
$$y$$ is too small. For example, if $$z$$ = 90, and we have 10 swaps for *p1*, and 10 swaps for *p2*,
 then in total we have 110 swaps plus whatever *p3* and *p4* give. Permutations of 6 can at most take 15 
swaps to order, while permutations of 7 at most 21 swaps, i.e., we cannot have the required
150 swaps (but only 146), 
so we can immediately advance the *p2* loop to the next permutation. One can do even better
by first ordering permutations by decreasing swap count. In that case, we can *terminate* loops
early because every other candidate that is left in the loop would contribute the same or fewer swaps to
the total as the one we are about to skip. With swap-ordered tables of permutations, the search becomes:
```python
threshhold = 150
swap0 = z
if z + 66 < threshhold: continue
for p1 in permutation6_table:
   swap1 = swap0 + p1.swaps
   if swap1 + 51 < threshhold: break
   for p2 in permutation6_table:
      swap2 = swap1 + p2.swaps
      if swap2 + 36 < threshhold: break
      for p3 in permutation6_table:
         swap3 = swap2 + p3.swaps
         if swap3 + 21 < threshhold: break
         for p4 in permutation7_table:
            swap4 = swap3 + p4.swaps
            if swap4 < threshhold: break
            cand2 = unsort(candidate, p1, p2, p3, p4)
            if is_13ordered(cand2):
               print(cand2)
```
(for simplicity, the optimization of 13-ordering checks is omitted above).


#### Generating permutations

The final task left is generating all permutations of 6 and 7 elements for the precalculation. 
I was lazy and simply lifted [Heap's algorithm](https://en.wikipedia.org/wiki/Heap%27s_algorithm)
from Wikipedia, in its nonrecursive form.


### End result

So what is the result after about 35 hours of running?
It was quite a cliffhanger - the first ~95% of the calculation barely does anything
because almost all the 160 million candidates are found and processed near the end
(in retrospect, stepping through counter 1 in reverse order would have been more pragmatic).
In the end, all candidates were accounted for, and when the dust settled, **the trial sequence**

* **[18,24,5,11,17,23,4,10,16,22,3,9,12,15,21,2,8,14,20,1,7,13,19,0,6]**

**found in Step 0 turned out to be actually the worst possible(!) 25-element sequence,
requiring 12 + 57 + 93 = 162 swaps to (13,4,1)-shell-sort.**

While no other sequence could beat this one,
the astute reader might notice that
there is a possibility that another sequence might also score the same 162 swaps. 
That is because 162 can be reached as 12 + 66 + 84, which our search would have missed
because we imposed $$z \ge 85$$. However, it turns out that $$y = 66$$ is impossible for
a 4-sort on an already 13-ordered sequence. To see that, consider the 13-ordering
conditions i) through iv)
for a maximally 4-unsorted sequence, i.e., when $$y = 66$$. This requires that the groups
6-6-6-7 are each in reverse order. Let 

* the group $$(4k+1)$$ be: $$a_1 > a_2 > a_3 > a_4 > a_5 > a_6$$

* the group $$(4k+2)$$ be: $$b_1 > b_2 > b_3 > b_4 > b_5 > b_6$$ 

* the group $$(4k+3)$$ be: $$c_1 > c_2 > c_3 > c_4 > c_5 > c_6$$

* the group $$(4k)$$ be: $$d_1 > d_2 > d_3 > d_4 > d_5 > d_6 > d_7$$

By condition iv) we have $$d_1 < a_4$$, so:

* $$a_1 > \{a_2, ... a_6, d_1, ... d_7\}$$,

where the set notation here means is that $$a_1$$ is bigger than any number in the set. Then, 
from condition iii) we have $$c_1 < d_4$$, so

* $$a_1 > \{a_2, ... a_6, c_1, ... c_6, d_1, ... d_7\}$$,

and from condition ii) we have $$b_1 < c_4$$, thus

* $$a_1 > \{a_2, ... a_6, b_1, ... b_6, c_1, ... c_6, d_1, ... d_7\}$$,

which means that $$a_1$$ is the largest element in the set of 25 numbers. However, that contradicts
condition i) that says $$a_1 < b_4$$.
Thus the sequence found above is unique - *every other permutation requires fewer than 162 swaps to
sort using (13,4,1) shellsort*.

*Q.E.D.*


#### Closing remark

What bugged me most about this problem is not that it is beyond undergrand level (I think it is)
but that the people who gave it have very likely no idea about what it actually takes to solve it. 
Of course, there might be some neat math shortcut to the solution but until I see it I am skeptical...

---

[back](/)
