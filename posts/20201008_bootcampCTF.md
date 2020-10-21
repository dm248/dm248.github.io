# b01lers bootcamp CTF

##### Oct 8, 2020


We hosted [bootcamp CTF](https://ctftime.org/event/1089) last weekend, This time I went for easy difficulty
and made a whole bunch of minichallenges (aka Crypto World). Things ran very smoothly, only a few
minor lessons to learn from.

The new CTFx-based infrastructure looked gorgeous and worked fabulously (as far as I know, it was 
novafacing on b01lers who practically did it all single-handedly). Moreover, to my surprise, all 
challenges could be handled just fine by a single 8 core, 32 GB RAM machine that we happened to grab 
at the last moment :)

I always wanted to make a text adventure game, so that is what gave the format for Crypto World. 
Programming-wise, the game itself is a surprisingly dull and quite error-prone endeavor with all its 
walls of text and area-specific conditionals. The client code was something I have never done before, 
and it annoyed me to no end that visuals seemed to depend on which browser I was using - until I got 
some local tips from elnardu on how to equalize things for Chrome and Firefox. Besides the scores of 
minichallenges, I also put two hidden challenges into Crypto World, in the hope that inaccessible 
areas will stoke people's curiosity (they sure would mine!).

In hindsight, a TODO list for my future 
self to read before the next event:

* **do more independent testing**. Actually we got lucky on this one because the day before nsnc 
caught some cut-and-paste printing bugs that affected a LVL 2 challenge or two (I am very thankful 
because to catch these you actually had to be serious and solve the respective LVL 1 challenge first). 
So the only real bug that remained in the minis was an unintended side-effect of a last-minute server hardening that 
capped the number of command arguments at 10 (which happened to be too few for one of the challenges).

* **have unit tests for too simple solutions**. The harder one of the two hidden challenges was 
actually borked during the CTF because a late refactoring in the server code removed one of the 
relevant checks. So the challenge got much easier than intended, and this got reported only once the 
CTF was over (I guess teams did not want us to roll back this juicier flag ;) I did test carefully 
each time that the intended solution did work, while the most naive attempts failed from the browser. 
But I forgot to check against direct naive attempts at the API :/

* **do not generate flags on the server, read flags from a separate file**. Automatic generation felt 
nifty but came to bite us when one of the challenges turned out to accept the super trivial 0, 0, 0 
solution. Rolling back all scores for it was easy but new flags also had to be created so that people 
would not just resubmit their old flags. So a quick and dirty patch was made in the server code 
to override the generated flags involved...

* **have accessible logs**. With the server running jolly well and lots of people wandering in Crypto 
World, I of course would have been curious to see who was trying what in there. I did have logging to 
stdout but there was no simple way to see those logs remotely. Having a secret diagnostics API will be
the way to go in cases like this.

* **watch out for Z3**. It was a surprise how much mileage people got out of Z3 on some of the 
challenges. This is something to keep in mind in the future (=make things harder for Z3).

* **have moderators in multiple time zones**. There were quite a few questions on Discord, some 
very basic ones too. But with everyone on the team in US timezone, there were hours when these had to go 
unanswered.


Overall, I am super pleased how well this CTF went.


---

[back](/)
