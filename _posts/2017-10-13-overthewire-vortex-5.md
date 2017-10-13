---
title: overthewire vortex level 5
---

# overthewire vortex level 5

after [solving vortex4](https://yuki-the-maven.github.io/2017/10/04/overthewire-vortex-4.html) I was ready for a new brainmelter. I was surprised to see that the challenge was *just* about getting and md5 right. and a little disappointed about the searchspace.

still, a brainmelter appeared and it was supereffective: I accidentally ended up being ridicolously hangover for the entire week, that does't stop me from solving challenges but also totally doesn't help with being smart at it.

## keys in space

I read md5, I think rainbow tables. I read

> a-z,A-Z,0-9 is the search space. The password length is 5 chars long

I go like gah, a search space like this will take like less than 10 minutes to bruteforce. not even fun? or am I missing something? doesn't look like there's a trick here.

## at the end of the rainbow

still the first impulse is to search for rainbow tables or use some md5-breaking functionality from oclhashcat or john the ripper. I fire up a kali vm and shell out some manpages and help. but despite the double dose of coffee my head is still pulsing so much from the previous night out that I don't really have the patience to scroll through to find what I might be looking for. I try duckduckgo instead but can't seem to find references to a md5-breaker. there's some mention of funcitonality that does so with rainbow tables but I would still need to download the tables first, so I start searching for that instead. I find some tables for :alphanum: 9 and 10 chars and they're at least 51GB in size. I need less characters and this is too big, it would take an entire year to get them over the abysmal wifi dongle we have in the office.

I lose patience, I lose focus, it's lunchtime and my colleagues are getting annoyed that it's the fifth time I said "just give me a sec, I'll start some tools and come for lunch…". whatever. I give up on learning something new for today. go grab a panini in a pathetic attempt to recover and move on to shellcast some bash violence.

## ugly force

so fine, search space is small, let's bruteforce it. I predict 10 minutes of cooking time and get some maths embarassingly wrong while trying to spill out the right formula for the space. luckily $colleague1 and $colleague2 are both having trouble remembering their math classess too and can't really make sense of my inconsequential rambling so embarassment averted.

the problem would be how to repeatedly feed input to the binary to see if the password matches: it asks for input with `getpass`, then either spawn a shell or prints a message and returns 0. no good for automating: we can't pipe to a `getpass` input, nor we have a clean way to tell when to stop. but well, we can just modify the provided c file to make our life easier! replace that `getpass` with `argv[1]` and the two if-branches with `exit(0);` and `exit(1);`.

now don't ask why, and this is the point of maximum wrongness, but I somewhat ended thinking that the least invasive modification of the binary would be best, left it at argv-exit and code the horriblest 5-nested bash loop ever on top of it. I even felt a little smart because `ps aux | grep a.out` would print the current invocation including the argument and that allowed me to track progress. I start it and try to get back to focus on work: something about resiliency, java libraries and rabbitmq.

## fast like a tree

fast worward three days after: the most significant char is still at `'n'` (iterating through lowercase first, then uppercase, then numbers in alphabetical order). waaaaat how could I've be so wrong in my guess of search time? also because of this I've been so naive that I spawned the 32-bit vm where I'm running this stuff with a single core. but now it's weekend so I'm home with access to better hardware. I quickly tweak the `Vagrantfile` I used for the previous level to run with 4 cores, partition the search space in and run 4 concurrent bruteforcers (plus the one still going on laptop).

a few hours later, still nothing and none of the scripts advanced by much at all. did I get the vm core settings wrong (*again*)? I `vagrant ssh` in and take a look at `top`. hm, at a glance it seems to do very inefficient use of the available processors. and, of course! at each attempt the shpell is spawning the binary anew, as a new process, with all the os overhead required to do so… to then perform a fairly inexpensive calculation and deallocate everything, throwing locality out of the window. this is as inefficient as it can be, and it's absolutely obvious… to someone sober.

## lighter than fast

tl;dr: I trashed the bash script, wrote the loop directly around the `main` and [this](https://cybre.space/@yuki_the_maven/4220638) was the result:

![mfw noticing that I was doing the most inefficient thing ever]({{ site.url }}/assets/vortex-bash-c-phail.png)

lessons learned: don't be permanently smashed, don't make decision about how to hack things in that state.

on to level 6 ¯\\\_(ツ)\_/¯
