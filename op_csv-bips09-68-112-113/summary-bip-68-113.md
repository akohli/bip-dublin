# Rough notes for tonight's meetup
http://www.meetup.com/Bitcoin-Dublin/events/231971651/


BIP113, BIP112 https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki 
BIP68 https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki Relevant PR https://github.com/bitcoin/bitcoin/pull/7910


# BIP 113
   
	- https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki
	- https://github.com/bitcoin/bitcoin/pull/6566/files
	  IsFinalTx() https://github.com/maaku/bitcoin/blob/dea8d21fc63e9f442299c97010e4740558f4f037/src/main.cpp#L641

	  https://github.com/maaku/bitcoin/blob/dea8d21fc63e9f442299c97010e4740558f4f037/src/main.cpp#L641
bool IsFinalTx(const CTransaction &tx, int nBlockHeight, int64_t nBlockTime)
{
    if (tx.nLockTime == 0)
        return true;
    if ((int64_t)tx.nLockTime < ((int64_t)tx.nLockTime < LOCKTIME_THRESHOLD ? (int64_t)nBlockHeight : nBlockTime))
        return true;
    BOOST_FOREACH(const CTxIn& txin, tx.vin)
        if (!txin.IsFinal())
            return false;
    return true;
}


calling:

```
bool CheckFinalTx(const CTransaction &tx, int flags)
{
    AssertLockHeld(cs_main);

    // By convention a negative value for flags indicates that the
    // current network-enforced consensus rules should be used. In
    // a future soft-fork scenario that would mean checking which
    // rules would be enforced for the next block and setting the
    // appropriate flags. At the present time no soft-forks are
    // scheduled, so no flags are set.
    flags = std::max(flags, 0);

    // CheckFinalTx() uses chainActive.Height()+1 to evaluate
    // nLockTime because when IsFinalTx() is called within
    // CBlock::AcceptBlock(), the height of the block *being*
    // evaluated is what is used. Thus if we want to know if a
    // transaction can be part of the *next* block, we need to call
    // IsFinalTx() with one more than chainActive.Height().
    const int nBlockHeight = chainActive.Height() + 1;

    // Timestamps on the other hand don't get any special treatment,
    // because we can't know what timestamp the next block will have,
    // and there aren't timestamp applications where it matters.
    // However this changes once median past time-locks are enforced:
    const int64_t nBlockTime = (flags & LOCKTIME_MEDIAN_TIME_PAST)
                             ? chainActive.Tip()->GetMedianTimePast()
                             : GetAdjustedTime();

    return IsFinalTx(tx, nBlockHeight, nBlockTime);
}
```


	  change:
	  
```
int64_t GetMedianTimePast(const CBlockIndex* pindex) {
	              int64_t pmedian[nMedianTimeSpan];
		      int64_t* pbegin = &pmedian[nMedianTimeSpan];
        	      int64_t* pend = &pmedian[nMedianTimeSpan];
        	      for (int i = 0; i < nMedianTimeSpan && pindex; i++, pindex = pindex->pprev)
             	      	  *(--pbegin) = pindex->GetBlockTime();
        	      std::sort(pbegin, pend);
		      return pbegin[(pend - pbegin)/2];
               }

```


# BIP 68
  - https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki
  - Relevant PR https://github.com/bitcoin/bitcoin/pull/7910

## Summary:
   Introduction of Relative Lock Time, RLT, for consensus enforced sequence number to remain invalid for a defined period of time after confirmation of its output.

   * repurpose of sequence number field, nSequence
   * see bip text for diagram of triggering this usage
   * time based relative clock, on 512s granularity
   * Walk through changes:
     - RLT change: consensus.h https://github.com/bitcoin/bitcoin/pull/7184/files#diff-cbe22f30d7e480617350ef6ceca97d0cR17
       and main.cpp https://github.com/bitcoin/bitcoin/pull/7184/files#diff-7ec3c68a81efff79b6ca22ac1f1eabbaR709
     - Flag impl: policy.h https://github.com/bitcoin/bitcoin/pull/7184/files#diff-e88cad3b27bf83e522e164d82186712dR46
       and transaction.cpp 

