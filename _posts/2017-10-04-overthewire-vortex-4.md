---
title: overthewire vortex level 4
---

# overthewire vortex level 4

I think it was 3 months ago.

after beating [overthewire vortex 3](http://overthewire.org/wargames/vortex/vortex3.html) ofc I couldn't stop myself and went straight to look what level 4 was about, and was presented with [this fairly cryptic code](http://overthewire.org/wargames/vortex/vortex4.html).

two immediate 'wtf's:
- wat? exit if any `argv` but then use `argv[3]`??
- oh cool I had no idea format strings are exploitable, let's see the reading material

and I opened [the first of the recommended readings](http://julianor.tripod.com/bc/NN-formats.txt). when I got to the part about `%n` there was a spike of excitement, immediately followed by a knot of confusion forming in my brain, like, bad enough that I semiunconsciously refused to think about it for 3 months (while still keeping a browser tab and a shell open on desktop #2 all way along). then last weekend I had this random impulse to pick it up again (idk, you know like when you need to spice up your boring not-many-chances-to-code-anymore life? no? good, it probably means that your $dayjob doesn't suck like mine (yet)), and it resulted in spending an entire sunday plus my flying time and lunchbreak on monday (and I had to look up hints! but more on this later). still, I eventually got there and this is a report of my journey of discovery and pwnage.

## goals

we're logged in as `vortex4`. need a shell as `vortex5` or to output `/etc/vortex_pass/vortex5`. the reading material explains how with the `%n` interpolation it's possible to write bytes in memory. last level we saw how the `.fini_array` pointers can be overwritten to point to shellcode we control, so the general idea would be:
- find out how to bypass the no-`argv` check
- pass some shellcode as an env
- use a format string to overwrite `.fini_array` with the address of our shellcode

## let's not argve about that

on the arg check: I don't want this to sound like the [how to draw an owl](https://imgur.com/gallery/RadSf) meme but I kinda just had this hunch and it worked right away: thinking about the signature of `execve` I used in vortex3, it suggests that `argv` and `env` might go close (or rather, literally next) to each other in memory. by virtue of being contiguous, addressing `argv[n]` of an empty `argv` actually overflows to `env`, but still keeps `argc` to 0. noice and easy! just write a small c wrapper that `execve`s with an empty `argv` and the fmt string we need in `env[2]` (note that both `argv` and `env` need to be `nullptr`-terminated). we'll mostly work with the wrapper from now on.

## write what?

let's grab some shellcode. the [execve shell in 24bytes available on exploitdb](https://www.exploit-db.com/exploits/42428/) is one of my favs now. tbh I first tried to generate something with `msfvenom` but I was confused by the fact that it got compressed with `shikata-ga-nai` by default and I couldn't clearly see the sled, then when I thought I got the exploit it just segfaulted: I decide to go for something more understandable and just add a very visible sled of `\x90` in front of it. at the very least I can rule out being an interference from getting the shellcode wrong. so, ~100 nops & 24 bytes of shell. let's smash them in a string and, uhm, let's point at it from `env[1]`, so right before our fmt string.

now, what to overwrite with the address of this? last level I used `.fini_array`: a quick combo of `objdump` and `readelf` reveals that it's still writable and its address is `0x08049f0c`. good let's go for that! now, we don't know the address of the shellcode nor the offsets of our strings from `$esp` and we need to balance both to hit.

## learn how2write

I won't go too much into how this works (there's no way I can do a better job at explaining this than the reading material) but to recap quickly:
- `%n` writes the number of bytes already written to a memory location which address is in the positional argument related to `%n`
- `%offset$n` allows to pick the write address from any offset
- `%x` can be used intead of `%n` to output bytes while testing for the offsets
- I seem to understand that `gdb` skews memory offsets so it's not reliable to get shellcode's address but it can give an indicative value maybe?
- endians are smol

at first I didn't get that thing about the `%.000u` modifiers mentioned in sloth's paper so I ended up creating a buffer sized 1024 and filling it with `A`s, then replace chars at strategic points with the `%n` sequences needed. we need to write 4 bytes, not all of them will be 255 so 1025 should be plenty of `A`s to generate any conbination of numbers we need to write.

now for how to write to the desired address: `%n` needs to refer to an address and writes to it. let's decompose the address at where `.fini_array` starts in 4 1-byte addresses and write them in `argv[3]` (remember endianness!): now if we can figure out the offset of these values from our "regular" `printf` arguments we can write `%offset$n` and target the pointers we just wrote as the desired writing destinations. now question is: where's our stuff? fire up `gdb`! `break main`, `run`,`x/2000x $esp` will stop at the beginning of main and dump 2000 words from the location pointed to by `esp` towards high memory: we are looking for a bunch of `\x90` (nop sled), followed by *something*, followed by a massive amount of `A`s, followed by the pointers to `.fini_array`. we easily (or if not just dump more words) find the sled and `A`s but the pointers look mangled, like cut and mixed: they're not aligned to words, and movements of `esp` and `ebp` tend to be aligned. that means that targetting an offset from `esp` will never hit the four bytes of our pointers but rather *in between* two pointers, rendering them useless.

## align of sight

immediate intuition: add bytes to the format string to make the bytes in the string right after (remember: format string is in `argv[2]`, pointers in `argv[3]`). so I change the size to 1025, recompile and refire gdb, only to find out that everything's as skewed as before. I try 1029 instead, thinking that it might need at least 4 bytes to change something, no result. but still I take a look at the memory around, and notice that it's `argv[1]` that is moving around!

we know that:
- allocations on the stack work by starting from a high memory address and subtracting bytes accordingly to the size needed
- `argv[n]` appears at lower memory addresses than `argv[n+1]`
- changing the size of `argv[n]` causes `argv[n-1]` to move

new intuition: stuff at the end of `argv` gets allocated first, aligned with a word mark, therefore *appending stuff at the end of `argv[3]` will cause the pointers to move*. in my case three `A`s did the trick.

## ascii or it didn't happen!

at this point I got real confused so let's try to draw what we know about the memory layout, for future reference. pen and paper are invaluable when learning this stuff, with pointers going back and forth it's real easy to get confused without an image.

it kinda looks like this (addresses are approximate to give a hunch of directions):

```
~0x08048154 ---- low memory, start of binary's addressing space
.
. various elf stuff, .got, .plt etc
.
0x08049f0c <--- .fini_array
.
~0x0804a024 <--- end of binary's addressing space
.
.
~0x0fff0000 <--- start of stack is around high memory
.
~0x0ffe1000 <--- $esp will be pointing around here when printf is called
.
.
~0x0fffc000 <--- argv
~0x0fffc001 <--- env[0]
~0x0fffc002 <--- env[1] (shellcode)
~0x0fffc100 <--- env[2] (fmtstring) "AA..AA%off1$nAA..AA%off2$nAA..AA%off3$nAA..AA%off4$n"
~0x0fffc002 <--- env[3] (address of fini) "0x08049f0c, 0x08049f0e, 0x08049f0e, 0x08049f0f", we need to point off1..4 in the fmtstring to here
.
.
~0x0fffffff <--- bottom of the available stackspace

```

next step will be to find the pointers' offsets from main's stackframe.

## scavenge the dump

in the alignment step we saw our strings being thousands of words away from `$esp`, but in reality they are not. why that is, I don't even. extra padding from gdb? maybe.

thing is, we have a way to dump memory (`%offset$x`) and will lazily whip up a templatized version of our c wrapper that we can recompile and rerun over and over again with different offsets, dumping memory until the values we want to point to. a [shpell](https://cybre.space/@yuki_the_maven/4290366) can look like this:
```
for i in $(seq 0 130);
  do cat wrapper.c.tpl \
  | sed "s/offset1/$(printf '%03d' $i)/" \
  > wrapper.c \
  && gcc wrapper.c \
  && echo ${i} \
  && ./a.out;
done
```
given that `wrapper.c.tpl` is a valid c program *minus* the offset that is a placeholder instead. this replaces the placeholder with values 0-130 (tweak as needed, one of my rounds ended up around 524) padded to three digits (remember you want to keep the string sizes static or you risk skewing addresses!), compiles and runs to dump the memory - along with the offset we're looking at for reference. serve cold with eyeballing or sprinkle with fresh `grep` before serving. you can do fancier things to dump more stuff in one go (like using `expr` and more palceholders to do bash math and go 4 by 4).

## point and creak

sweet, we have our offsets. so now replace in the wrapper, change those `%x` to be `%n` once more and fire! boom, segfault. everything looks fine. it would be great to be able to see a core dump to understand what went wrong but I couldn't find a way to get one on the server. it might be the offsets. it might be that my shellcode has something invalid in it. it might be that I'm overwriting the address of `.fini_array` with an invalid memory location (at this point I've not yet understood fully the thing about `%.001u` and wrapping at 256 - do read it carefully, the examples are a bit obscure because they're demonstrated with easyflow but it's important). in a deperate attempt to reduce complexity this is were I replaced the paylod from `msfvenom` with the simpler one from exploitdb+sled. still broken. I move on a local vagrant box and repeat the steps with the same code (offsets will be different so I need to figure them out again). boom again. but here I can `ulimit -c unlimited` and read a coredump!

and the dump reveals that the segfault happens in `vprintf`, and all the values are as we would like them to be. so what's wrong?

## a note on compiling

server machine is a 32-bit with aslr disabled. latency tends to be quite high so I'd recommend working locally on a similar environment for experimenting (then ofc the actual offsets will need to be calculated on the actual game server). for this purpose I used [vagrant](https://www.vagrantup.com/) with the [hashicorp/precise32](https://app.vagrantup.com/hashicorp/boxes/precise32) box

this vagrantfile can help bootstrap (implies having a copy of the `vortex4` binary):
```
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise32"
  config.vm.box_version = "1.0.0"
  config.vm.provision "file", source: "vortex4", destination: "vortex4"
  config.vm.provision "shell", inline: "apt-get update && apt-get install -y gdb vim"
  config.vm.provision "shell", inline: "mkdir -p /vortex"
  config.vm.provision "shell", inline: "mv vortex4 /vortex/vortex4"
  config.vm.provision "shell", inline: "chown root. /vortex/vortex4"
  config.vm.provision "shell", inline: "chmod +u /vortex/vortex4"
  config.vm.provision "shell", inline: "echo 0 > /proc/sys/kernel/randomize_va_space"
end

```

not precisely overthewire's server but easily available and low effort in bootstrapping

fun fact: when I first used this I forgot to disable aslr and tried to do the offset calculation. you can imagine how well that went.

## give up

banged my head on this for a couple of more hours, decided to have some sleepz. obviously that didn't go very well and I searched for writeups instead. shameful, but I had a hunch: if level 3 was recompiled with different options, can the same have happened to level 4? I can probably scroll the posts quick enough to avoid major spoilers and read only comments and the specific steps I'm stuck at. boom! someone in the comments suggests "recompiled, partial relro, do else". wtf relro even is? a quick duckduckwalk reveals that it's a way to declare parts of memory as not writable, even if `readelf` (and by extension me) thinks they are. whatever, same comment also suggests that `exit` in `.got` is still writable (yup, couldn't escape *all* the spoilers) so let's go for that instead of `.fini_array`. tbh that was the first guess I had at what to overwrite for level 3 as I read about it in [a bug hunter's diary](https://www.nostarch.com/bughunter) but then decided not to in favour of following the hints. ohwell.

## exit

I'll spare you the part in which while changing the addresses to point at `exit@.got` I get the offsets wrong again, get annoyed and decide to stop and simplify the wrapper (bigger sled, smaller fmtstring, use the `%.001u` to better control which bytes to write, move the pointers at the beginning of `argv[2]` for less moving parts…).

tl;dr: attacking `.got` isntead of `.fini_array` worked perfectly, and spawned a sweet sweet shell as `vortex5`!

that was mind-bending! but got there eventually. ofc I jumped on level 5 right away, but that's a story for tomorrow.
