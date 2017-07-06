---
title: overthewire vortex levels 0-3
---

# overthewire vortex levels 0-3

so I finally just beat [overthewire vortex 3](http://overthewire.org/wargames/vortex/vortex3.html) and thought it would be cool to write down how I got there knowing only how to do the "control the ret address thing".

I'll quickly go through 0-2 too, don't have anything incredibly exciting to say about them tho.

## vortex0

have to write something that reads few bytes from a port and writes something back. instructions tell right away that you need to keep endianness in mind. fairly trivial.

I decided to go for golang as my language of choice as I'd like to learn more of it, I don't like python much and I tend to see golang as a possible decent alternative to it. ugly af, can manipulate bytes easily, comprehensive stdlib, easily accessible docs.

a golang solution would likely revolve around a `net.Dial("tcp", "vortex.labs.overthewire.org:5842")`.

## vortex1

the sauce contains a lovely macro that will kindly drop a privileged shell for us if `ptr` has a specific value in the most significant byte. we can decrement the value of ptr by writing more `\` characters in the program, but it starts very far from the desired value. I started making a file with enough `\` chars to use as input and ended up being several megs, at that point I realized that, well, actually, `ptr` being on the stack next to the `buf` it points to, I might as well have it point to itself and overwrite with the desired value. backspace insanity successfully dodged.

there also needs to be away to keep stdin/out open or the shell closes right away, something like this will do: `( ./command-that-generates-program-stdin-input-generally-a-py-script ; cat) | /vortex/vortex1`

## vortex2

well, this level will generate a nice tar for us. with the flag inside with the right params. I got somehow confused about that being written in `/tmp` as I thought it was set to not writable but was fine. then got confused about what that `$$` was about, thought it was some tar placeholder that can be set with cli options. when I realized what it was I started throwing around various `find -name` replacing that `$$` with probable pids and ofc got nothing because I totally forgot that we're looking at a c program and not a bash one so `$$` is just itself in that string. I felt like a total idiot for that but in my defense it was fairly late after a v long day.

## vortex3

open the instructions page, spot the buffer, spot the vulnerable `strcpy`… waitwhat there's no function call?

I know how to do a basic ret-address-hijack and redirect it to a `getFlag` function that is provided in the binary. I struggled a lot during my recent go at [behemoth1](http://overthewire.org/wargames/behemoth/behemoth1.html) and ultimately learnt about shellcode and how you kinda need a nopsled for that. but I look at this and I'm, like, wtf?

the instructions are hinting at stackshield, stackguard and dtors. so I try to get on the linked phrack articles, just to discover that my mobile carrier censors them. luckily I have a local copy (`wget -r` ftw). I skim through stackguard/shield but I fail to see how that's related to the problem at hand, that is I don't have a ret to hijack. also what does that have to do with dtors? I assumed dtors were related to stackguard but it doesn't seem so… let's investigate that.

incidentally by reading phrack and some articles on poc&#124;&#124;gtfo I got some extra insight in elf and commands such as `readelf` to figure out sections and what's writable. so what about these dtors? well, they're not there at all. what's writable? these `.init_array` and `.fini_array`, whatever they are. I still don't get it and keep tinkering with `objdump` for a few days.

first thing I actually did after realizing that there is no ret was to check for casts to function pointer but there are none so I started ignoring the code and look into all the above because I was concerned about this disturbing lack of ret. but after tinkering with `objdump` and going to the code again, somthing caught my eye right away: the condition for that `exit(2)` is clearly referencing the addressing space of the image. good hint! also, that pointer mess… it's ultimately making *something controllable* point to the controllable buffer. noice.

hypotesis:
- shellcode goes in buffer
- overflow buffer, overwrite lpp with *something writable in the image that is also a pointer and is executed at some point*
- profit

I still need *something writable in the image that is also a pointer and is executed at some point*, time to look up wtf that `.fini_array` is. aaaaaaand sounds like bingo. cleanup stuff added by the linker, it's an array of fnpointers to cleanup fns executed at exit time.

I know the size of the buffer, I know that `lpp` is a few bytes above, it's late and I have no intention to do hex math so the smart idea is to write a program in golang that:
- runs the program with 128 'A', this returns 0
- repeats with increments of 'AA' at a time
- when the run returns 2 it means we overwrote `lpp`'s most significant bytes so current input length +2 is the right overflow size.

easy, right? well, [this](https://cybre.space/@yuki_the_maven/2031300): ![mfw failing at golang]({{ site.url }}/assets/vortex-golang-phail.png)

but I get there. welp, all this hassle to avoid literally a sum in hex. anyway, now I have a lovely golang buffer-size-finder, time to replace those 'A' with some shelly goodness. last time for behemoth I picked a 23 byte drop-a-shell from [exploit database](https://www.exploit-db.com/) but this time I'll try `msfvenom` to get familiar with it. so I start my kali vm, check `msfvenom --help`, find the payload listing and look for a shell dropper: I expected a small, standard just-execve-bash for x86 to be there but nope, there is a more flexible one. I's still have to modify it to keep the setuid privilege so I decide to go for a file reader instead (I just need to read `/etc/vortex_pass/vortex4` after all, why bother with a shell and privs?). `msfvenom` automatically gives it a few round of `shikata_ga_nai`. out size is 111 so fine by me.

I stick the shellcode at the beginning of the buffer, replace the last 4 bytes with the address of `.fini_array` and start my go toy and… nothing happens, exit 0. wat? quick check at the payload… yep, it contains null bytes. `msfvenom` has an option to specify the bad bytes but I assumed that `\x00` and `\xff` were out by default. they're not. go get myself a sanitized payload. drum roll… segfault. it's late, so I just crash to sleep. for two more days I stare at that code (while annoying half of my team with the whole story) and poke with `gdb` to find out where exactly the segfault happens. is that `.fini_array` actually not writable? or am I missing something? well yes, the patience to read stuff carefully: `lpp` is dereferenced twice so if I set it to point to that array what I'm actually overwriting is the cleanup function, not the pointer to it. and that's ofc not writable.

so what now? how do I get a pointer to `.fini_array`? because of the check above that `exit(2)` it must still be in the image. does it have something to do with that weird `tmp` juggling? hmm, looks more like a red herring. I go back to `objdump -Ssd` and search for `.fini_array`'s address. no hit. but wait, endianness! search for the same in reverse order… hit! I take note of that and replace it in my overwrite. kaboom! it explodes spectacularly with a different segfault that dumps shellcode and env altogether, but I suspect that's because I'm testing on my machine. so I jump over the wire and inject the same bytes.

the result is the content of `/etc/vortex_pass/vortex4`, printed perfect clean on the console.

## next

will continue this, and hopefully write more about the journey! this stuff is so much fun I can't stop doing it and telling everyone about it!
