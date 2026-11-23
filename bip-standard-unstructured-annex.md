```
  BIP:
  Layer: Applications
  Title: Standard Unstructured Taproot Annex
  Authors: Peter Todd <pete@petertodd.org>
  Status: Draft
  Type: Specification
  License: CC0-1.0
```

## Abstract

This document specifies a standardized way to use taproot annexes for
unstructured, non-consensus, data. With the aim of being compatible with future
consensus-meaningful uses of taproot annexes.


## Motivation

The taproot soft-fork defined the taproot annex as an optional, per-input,
field in each transaction input witness of unlimited size and no consensus
meaning. Signatures in a taproot transaction input sign the contents of the
annex in that txin; signatures do *not* sign the annexes of any other
transaction input.

For applications that need to provably publish non-consensus data in the
Bitcoin chain, using the taproot annex has many desirable advantages over other
techniques in many circumstances:

1. Cost: since the annex is in witness data, it has a per-byte cost 1/4th of
   `OP_Return` data.
2. Simplicity: while there are other ways to publish data in input witness
   space, such as commit/reveal schemes, the annex is the simplest from a
   technical perspective.
3. Natural, per-input, association: since each input has an annex, publishing
   data in the annex has a natural meaning of being directly associated with
   the at of spending that output.

Conversely, these same advantages are equally applicable to future consensus
upgrades.

FIXME: specific examples


## Specification

A transaction input containing a taproot annex is considered standard if the
annex is either:

1. Empty.
2. OR starts with the null byte, 0x00.

(Optional, to mitigate transaction pinning attacks) A transaction is considered
standard if:

1. Every transaction input contains an annex (with the exception of [P2A](https://github.com/bitcoin/bips/blob/master/bip-0433.mediawiki)
   spends).
2. OR no transaction input contains an annex.


### Transaction Pinning Attack Mitigation

Since the annex is only signed by the signature(s) in the input it is contained
in, allowing the annex to be used creates a new type of transaction pinning
attack: in a multi-party transaction, a participant can great a large,
high-fee/low-fee-rate, version of the transaction by resigning their input with
a much larger annex. Requiring all inputs to explicitly opt-in to annexes
prevents this issue from affecting non-annex-using protocols; for annex-using
protocols TRUC is an option to partially mitigate transaction pinning.

This mitigation can be removed once both replace-by-fee-rate, and witness-only
replacement, is implemented, allowing the smaller, high-fee-rate, witness
variant to simply replace the pinning transaction.


### Forward Compatibility With Future Consensus Upgrades

Here we need to imagine what future consensus uses of the annex could look like
to ensure that we are leaving design space for them. For prior art we have
[BIP-68: Relative lock-time using consensus-enforced sequence
numbers](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki), the
only soft-fork to (so far) assign a totally new[^bip-113] consensus meaning to
an existing transaction field. BIP-68 avoided affecting *standard* transactions
at the time by only enabling its behavior for transactions with `nVersion >= 2`.

[^bip-113]: [BIP-113: Median time-past as endpoint for lock-time calculations](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki), *modified* the meaning of an existing transaction field, `nLockTime`, by slightly delaying time comparison to the median-time-past rather than the `nTime` field in the block header. Unlike BIP-68, this change also affected standard transactions, as it applied to existing uses of time-locking.

One approach would be to just assume that any future use of the annex would
also be accompanied by a `nVersion` activation. With that assumption, we could
just make use of the annex standard, without requiring the null byte.
Unfortunately, [BIP-431: Topology Restrictions for
Pinning](https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki) (TRUC)
throws a wrench in that idea: it uses `nVersion=3` for special size and
topology limiting behavior that clearly won't be desired by all transactions,
and it's quite unclear how that behavior will be adapted in the event that new
`nVersion` consensus meaning is adopted.

Let's suppose that a future soft-fork *does* adopt new `nVersion` functionality
related to the annex: the proposed unstructured annex format will be forward
compatible, as we are not proposing to change standardness rules around
`nVersion`.

On the other hand, if a future soft-fork adopts new `nVersion` functionality
via the format of the annex itself, we've given significant freedom to do so:
just adopt a non-empty format whose leading byte does not start with 0x00.

### Delta-Tag-Length-Value Encoding

To give an example of how a soft-fork could adopt new annex functionality in a
way that is compatible with the unstructured annex proposal, consider the
delta-tag-length-value encoding:

    <delta-encoded tag> <varint length> <tag payload>

Here, each tag would be encoded as the *delta* of its tag number and the
previous tag, preventing the serialization of redundant tags. A zero-delta
naturally maps to the concept of "no more tags".


### Forward Compatibility With Future Mempool Policy/Standardness Features

We might want to use the annex to signal non-consensus, mempool policy choices.
For example, an alternative way to implement TRUC would have been to put a
signed, non-consensus, mempool policy hint, in the annex specifying the maximum
transaction size[^segwit-annex]. Again, our unstructured annex proposal is
forward compatible with this use, as it could be encoded with a
(delta)-tag-length-value scheme allowing both to coexist.

[^segwit-annex]: Note how the segwit soft-fork could have added a per-transaction annex, signed by all inputs, which would have naturally enabled functionality like this without per-input duplication. A missed opportunity!


## Reference Implementation

https://github.com/bitcoin/bitcoin/pull/36466


## Copyright

This BIP is licensed under the CC0-1.0 license.
