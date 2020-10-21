# b01lers bootcamp CTF

##### Oct 8, 2020


We hosted [bootcamp CTF](https://ctftime.org/event/1089) last weekend, This time I kept the difficulty
easy and made a whole bunch of minichallenges (aka Crypto World). It all went very smoothly, only a few
minor lessons to learn from.

The new CTFx-based infrastructure looked gorgeous and worked fabulously (afaik it was practically one 
person on b01lers who did it single-handedly). To my surprise, all challenges could be handled just 
fine by a single 8 core, 32 GB RAM machine that we happenned to grab at the last moment :)

I always wanted to make a text adventure game, so that is what gave the format for Crypto World. 
Programmatically, the game itself is a surprisingly dull and quite error-prone endevour with all its 
walls of text and special conditions. The client code was something I have never done before, and it 
annoyed me to no end that visuals seemed to depend on which browser I was using - until I got some 
local help on how to equalize things for Chrome and Firefox. In hindsight, a TODO list for my future 
self to read before the next event:

* *do not generate flags on the server, read flags from a separate file*. 

Automatic generation felt
nifty but came to bite us when one of the challenges turned out to have a super trivial solution.
Rolling back all scores for it was trivial but new flags also had to be created so that people would
not just resubmit their old flags (so there is a quick and dirty patch in the server to override the
generated flag...). 


* 



---

[back](/)
