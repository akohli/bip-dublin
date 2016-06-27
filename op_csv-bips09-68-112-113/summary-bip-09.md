# BIP 9 - Verson bits with timeout and delay

https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki

## Goals
 - multiple deployments
 - recovery of locked in bits
 - less coordination between developers
 - Replaces BIP34 (IsSuperMajority mechanism)
 - ISM downsides: 
   - Requires tallying every block (load the past 1000 blocks every block)
   - reduces available bits for use! It was originally a signed int, but BIP34 states version >= 2
     2^32  [>1]: 2^31-1
   - only one at a time, competition over what gets voted on first
   - never had a timeout period, technically should always be checked
   
## Specification: 

The period over which bits are tallied is currently the difficulty retarget interval. (BIP34 had 1000 blocks) 

This means the state of a deployment remains unchanged for at least 2 weeks at a time (2016 blocks). 

The threshold at which a rule is accepted is currently 95% (unchanged from BIP34). 1916 blocks
during a period will have to support a deployment in order for it to be accepted. 

### Deployment: 
 - defined: `name`, `bit`, `start time`, `timeout`

CSV: https://github.com/bitcoin/bitcoin/blob/master/src/chainparams.cpp#L90
Segwit: https://github.com/bitcoin/bitcoin/blob/master/src/chainparams.cpp#L95
 
### Interpreting block flags: 

The field is now a bit-field. The available space is limited by the previous rules (34, 65, 66)
The most severe: since the numeric version is signed, we can't use the left-most bit ever!
We specify that the bits '001' must be found at the start of the version: 
Base:       0010 0000 | 0000 0000 | 0000 0000 | 0000 0000 
  
  The lowest version with no bits set: 
    0x 20 00 00 00 (536870912 in decimal)
    
  The version if all bits are in use: 
    0x 3F FF FF FF (1073741823 in decimal)
    
This also avoids collision with previously locked in versions, (2, 3, 4).

CSV:        0010 0000 | 0000 0000 | 0000 0000 | 0000 0001
Segwit:     0010 0000 | 0000 0000 | 0000 0000 | 0000 0010
  
### Warnings: 
Expect deployment bit CSV/Segwit, but observe bit 3
        
Bad:        0010 0000 | 0000 0000 | 0000 0000 | 0000 0111
        
Our  :      0010 0000 | 0000 0000 | 0000 0000 | 0000 0011
~Ours:      1101 1111 | 1111 1111 | 1111 1111 | 1111 1100

block.version & ~ours: 
            0000 0000 | 0000 0000 | 0000 0000 | 0000 0100
            
This is not equal to zero, and indicates a block was mined with support for a deployment
we don't know about.

Core seems to do this on a bit level:
https://github.com/bitcoin/bitcoin/blob/master/src/main.cpp#L2322
by checking a bit is set in the version, and not set in the result of ComputeBlockVersion

### States:
*DEFINED* -> *STARTED* -> *LOCKED IN* -> *ACTIVE*
                                     [-> *FAILED*]
                                     
- States are calculated every 2016 blocks (the retarget interval)
- Every 2016th block, the state is recalculated starting from `block.height - 2016`
- During the *STARTED* phase, miners will vote deciding whether to set the bit defined in the deployment.  
 
State calculation: 
https://github.com/bitcoin/bitcoin/blob/master/src/versionbits.cpp#L23

Rule activation:
https://github.com/bitcoin/bitcoin/blob/master/src/main.cpp#L2447
 
### New rule activation:

  If the bit-count exceeds the threshold over the phase, it will be bumped to *LOCKED IN*
  During a soft-fork, there is usually some testing to see if miners and clients correctly
  implemented the new rule. The *LOCKED IN* state allows for buggy software to be detected,
  such as SPV miners. Wallets can also suffer during a switchover should miners mine blocks
  according to older rules. 
   
  The *LOCKED IN* state presents an opportunity for developers to fix their software,
  before the rule is promoted to *ACTIVE*, where not implementing the rules may cause financial losses.
   
### Outlook:

BIP9 operates with 29 bits, allowing 29 of such deployments to be in the *STARTED* or *LOCKED IN* state
 
They bits themselves are all recoverable, so there is no risk of them running out. 

There might be contention over a bit if over 29 soft-forks are proposed at any one time. But that's
much better than trying to organize people into a one-by-one deployment process. 

The lifetime of a deployment is maximum should it enter *STARTED* and eventually reach *FAILED*
The minimum deployment lifecycle would be a single phase in the *STARTED*, followed by another in 
*LOCKED IN* - 4032 blocks or approximately one month. 

#### CSV:

CSV entered *STARTED* on the May 1st.
Bit: 0
Version is `0x20000001`
 
http://bitcoin.sipa.be/ver9-10k.png
> Graph with uptake of BIP9 flags

It reached *LOCKED IN* on 417312 (June 21st, 6am)
It will be *ACTIVE* on 419328 (July 5th, 6am)
 
 
#### Segwit: 
Bit: 1
Version is `0x20000002`

Segwit's start time is currently undecided, but the code has already been merged to Core.
It is not expected to make the `0.13` release date, though since it's a soft-fork we should
see it reach `13.1`. It's hard to know when this will be, but Core is often driven to a release
if enough users and miners run the code through an official release.