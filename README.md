# ps4 `rtsold`/CVE-2025-14558 exploits

## I have not achieved code execution on a PS4.
Not on Orbis 13.02, not on anything.
If you're here from some "PS4 JB" retweet, fuck you. I did what I could here, but its only research. I passed a POC to some trusted sources to iterate on, do not hassle them for it plz and thx.

## TL;DR

CVE-2025-14558 is real, and it's interesting i guess lmfao
FreeBSD's `rtsol(8)` / `rtsold(8)` pull the
DNSSL (domain search list) option out of an IPv6 router advertisement and hand it to `resolvconf(8)`
without validating it. `resolvconf` is a shell script. You can malform a packet and get userland execution, supposedly.

Orbis is FreeBSD-derived and the PS4 speaks IPv6, so "does 13.02 ship a `rtsold` that parses DNSSL and
feeds it to something shell-shaped" is a gud question. It is, however, a question. "FreeBSD-derived" is
a reason to look, not a result. NVD's affected-products list says FreeBSD, period. nothing about
Sony were mentioned, but I made this in the good faith of it being under something that may be affected atleast without an upstream patch. 


What I actually have:

- packet builders for a hand-rolled RA carrying a poisoned DNSSL option
- probes and diagnostics of varied sucess
- a stager and a payload set, but probably wont function

What I don't have: 

a single console-side artifact showing that a PS4 parses the option, runs this code
path, or executes anything I put in it. The gap between "scapy sent the packet" and "the console ran my
command" is the issue im facing here. **I have no way to actually test if it works** with what limited resources and funding i have.


## Claims n shit

Seperate these thoughts and claims, conflating them is how SKIDDING and HOOLIGANISM happens:

grrr >:c


| claim | where it stands |
| --- | --- |
| FreeBSD `rtsold`/`resolvconf` flaw exists | yes this hinges on it |
| poisoned DNSSL reaches the vulnerable path on stock FreeBSD | probably |
| Orbis 13.02 contains this code | fuck if i know |
| Orbis executes attacker commands through it | no, atleast from what ive tried |
| it'd be root-equivalent | depends entirely on what identity `rtsold` runs under and how Orbis sandboxes it. could be FAT fucking nothingburger, could be a somethingburger..... |

## the stuff i tossed together in an attempt to test it

**`ps4_exploit_fixed.py`** — hand-rolled RFC 1035 label encoding, manual DNSSL option construction, full
ethernet/IPv6/ICMPv6 RA sent to `ff02::1`, direct command mode, reverse-shell selection, marker-file
"check" mode. Two known warts: the EUI-64 builder doesn't flip the U/L bit, and the DNSSL bytes are
bolted on as a `Raw` layer whether any target actually parses the option the way I intend is
unverified. Also, when this prints "success," what it means is "the packet left the NIC."

**`ps4_exploit_full.py`** — structured stuff: direct payload, tiny HTTP stage-2 server, fetch/wget
staging, auto mode, interactive console, logging. Note that the "PS4 behavior model" in the source is a
model — i.e., something I wrote down, not something I observed. Don't cite it at me.

**`ps4_exploit_poc.py`** — mostly a copy of `full`. This is the only thing i actually released to anyone ever, if you can figure out who has it and somehow squeeze it out of them congrats, you get brownie points.

**`ps4_rtsold_detect.py`** — the honest one, ironically. `probe_ps4_dns_change()` is a `pass`
placeholder, and `detect_ps4s()` prints "Detection complete" and returns `[]`. The whole detection
story bottoms out in "go look at the target yourself." Naming it "detect" was bullshit and we both know it. Never leaving my personal pc, probably forever sob skull emoji.

**`ps4_rtsold_diag.py`** — EUI-64 conversion, RDNSS probe, a DNSSL probe with an observable side
effect, multi-RA file staging, a FreeBSD VM validation guide, full-stack diagnostic flow. RDNSS traffic
tells you the RA/DNS path is alive; it does not tell you injection works. Some of these probes will
change resolver config on a vulnerable box. Never released.

**`ps4_rtsold_stager.py`** — stage-1 generation, stage-2 templates, HTTP server, `--auto`. Heads up:
`--auto` stands up the server and prints stage 1 but never actually *sends* stage 1. The "all-in-one"
flow isn't. Several templates also assume tools that may simply not exist on Orbis. Also never ever seeing the light of day

**`ps4_debug_payloads.sh`** — debugging payloads for testing thingiez i tried. never seeing the light of day

**`ps4_payloads.sh`** — reverse shells, system survey, DNS exfil, memory-dump attempt, update
blocking, persistent tunnel. Post-exploitation material. Never gonna be released to anyone ever

**`ps4_shell.sh`** — interactive post-exploitation menu. Again, never finna be released.

***Kill yourself, my code will never see the light of day!!!!!!!!!!*** - Me if i were evil or smthn

:D
## the chainz

1. attacker on the local link fires an unsolicited ICMPv6 router advertisement at `ff02::1`
2. the RA carries a DNSSL option (type 31), DNS-label encoded, padded to 8-octet units
3. the target's RA client accepts it and hands it to resolver config
4. *if* the target's `rtsold`/`resolvconf` is the vulnerable pair, shell metacharacters in the option
   data get interpreted
5. impact = whatever that process can do

The repo implements steps 1 and 2. Steps 3–5 are the actual research question, and the repo does not
answer them

Protocol trivia I burned time on so you don't have to: hop limit 255 per ND; a single DNS label tops
out at 63 octets, which after the wrapper the code uses leaves roughly 61 bytes of payload. Hence
staging: short fetch command in the option, real script over HTTP.

## if it turns out real

Attack position would be network-adjacent, unauthenticated, at the IPv6 link layer. The fix is to validate DNSSL labels against a strict grammar before they touch anything, stop pushing network data through shell scripts, quote every externally-sourced field at every shell boundary, reject overlong labels / control chars / metacharacters, least-privilege the RA/DNS config service, and regression-test malicious DNSSL input.

also, this is almost certainly patched in 13.04, due to automatic upstream patches every firmware release ;)

## refs

- NVD: <https://nvd.nist.gov/vuln/detail/CVE-2025-14558>
- CVE record: <https://cve.org/CVERecord?id=CVE-2025-14558>
- RFC 4861 (ND), RFC 6106 (RA options for DNS), RFC 1035 (domain wire format)

**PS;** If a exploit ever does get made pl0x call it "GRANDE" for 'general router advertisement network daemon exploit' and PLEASE credit me in the research thx xoxo

I hope you find joy,

— bel / [@belsploit](https://twitter.com/belsploit) / <https://jb.0d01.wtf>
