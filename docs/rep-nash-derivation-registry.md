# REP-XXXX - Reddcoin BIP32 derivation path registry

```
  REP: (unassigned)
  Layer: Applications
  Title: Reddcoin BIP32 derivation path registry
  Authors: CryptoGnasher <gnasher@reddcoin.com>
  Status: Draft
  Type: Informational
  Assigned: (pending)
  License: MIT
```

- **Scope:** documentation and process. **No consensus impact**, no code change, no behaviour
  change. This REP records what Reddcoin software already derives and states the rule for
  assigning anything new.
- **Registers:** purpose `1018'` for release-manifest signing (§4.1); records the BIP-defined
  purposes already in use.
- **Relates:** the release-signing REP, which specifies the scheme under `1018'`.
- **Does not:** reserve any value outside Reddcoin. See §6, which is the most important section
  in this document.

---

## 1. Abstract

BIP43 introduced the purpose field but deliberately left its assignment to two mechanisms,
neither of which fits a coin-specific scheme in a fork like Reddcoin. The result is that
projects pick values by convention and record them wherever they happen to be implementing.

This REP is the one place Reddcoin records which BIP32 derivation paths its software uses, and
the procedure for adding to that list. It is a registry in the sense that SLIP-0044 is a
registry of coin types: a document that says what is in use, maintained because the alternative
is that nobody knows.

It makes no claim on any value outside this project, and says so explicitly, because a registry
that overstates its own authority is worse than none.

## 2. Motivation

### 2.1 The immediate cause

The release-signing scheme needed a purpose value. The first one chosen, `1017'`, turned out to
belong to lnd, which derives every Lightning key under `m/1017'/coinType'/keyFamily'/0/index`.
Nothing would have collided, because that key comes from a dedicated seed, but the path
described the key as something it was not.

The value was picked without checking, and there was nowhere to check. That is the gap this
document closes.

### 2.2 Why this recurs

Reddcoin's roadmap includes ReddID identity keys and social tipping, both of which plausibly
need key material at a path of their own. Each will be specified in its own REP, and each
author will face the same question with the same absence of an answer. Without a registry the
natural move is to pick a number that looks unused, which is exactly the move that produced
`1017'`.

### 2.3 Why a local registry rather than nothing

A registry cannot stop an unrelated project taking the same number. What it can do:

- prevent Reddcoin colliding with **itself** as schemes accumulate, which is the collision that
  actually matters, because it is the one that would break real wallets;
- give a reviewer a single file to check when a REP proposes a new path;
- record the reasoning behind each assignment, so a future maintainer inherits the decision
  rather than re-deriving it;
- make the assignment procedure explicit, so "check before you take" is a written step rather
  than a thing a careful person happens to do.

## 3. The purpose field

The governing text is BIP43, reproduced here so a reader has the rule in front of them rather
than a paraphrase of it. Quoted from
[BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) by Marek Palatinus and
Pavol Rusnak, Status: Deployed, Assigned 2014-04-24:

> We propose the first level of BIP32 tree structure to be used as "purpose".
> This purpose determines the further structure beneath this node.
>
> ```
> m / purpose' / *
> ```
>
> Apostrophe indicates that BIP32 hardened derivation is used.
>
> We encourage different schemes to apply for assigning a separate BIP number
> and use the same number for purpose field, so addresses won't be generated
> from overlapping BIP32 spaces.
>
> Purpose codes from 10001 to 19999 are reserved for [SLIPs](https://github.com/satoshilabs/slips).
>
> Example: Scheme described in BIP44 should use 44' (or 0x8000002C) as purpose.
>
> Note that m / 0' / * is already taken by BIP32 (default account), which
> preceded this BIP.
>
> Not all wallets may want to support the full range of features and possibilities
> described in these BIPs. Instead of choosing arbitrary subset of defined features
> and calling themselves BIPxx compatible, we suggest that software which needs
> only a limited structure should describe such structure in another BIP and use
> different "purpose" value.

BIP43 carries no copyright or licence section, predating the requirement for one. The passage
above is quoted as an excerpt with attribution. It is not relicensed under this document's MIT
licence, and the MIT grant in §9 applies only to the original material in this REP.

### 3.1 What this leaves for Reddcoin

BIP43 offers two routes to a purpose value:

1. **Apply for a BIP number** and use it. Appropriate for a scheme of general interest to
   Bitcoin. A Reddcoin-only scheme is not BIP material, so this route is closed in practice.
2. **The SLIP range, 10001 to 19999**, reserved for SatoshiLabs. Open to Reddcoin via the SLIP
   process, at the cost of a third-party review and wait.

Neither is a registry of values already taken outside those two routes, and a large body of
deployed software sits outside both: lnd at `1017'`, Cardano at `1852'`, Solana at `501'`.
Notably, SLIP-0013 itself uses `m/13'`, inside the BIP-number space rather than the range
reserved for SLIPs.

Reddcoin therefore operates the way the rest of that group does, and this REP records the
consequences honestly rather than implying an allocation that does not exist (§6).

## 4. Registry

### 4.1 Purpose values

| Purpose | Scheme | Defined by | Status |
| --- | --- | --- | --- |
| `0'` | BIP32 default account, used by the legacy HD wallet | BIP32 | In use, legacy |
| `44'` | HD wallet, legacy and descriptor | BIP44 | In use |
| `49'` | P2SH-SegWit descriptor wallets | BIP49 | In use |
| `84'` | Native SegWit descriptor wallets | BIP84 | In use |
| `1018'` | Release manifest signing | This REP §4.5, scheme specified by the release-signing REP | Self-allocated, §5.2 |

`86'` (taproot descriptors) is **not** in use: `SetupDescriptorScriptPubKeyMans` asserts on
`OutputType::BECH32M`. It is listed here only so a future implementer knows the value is
already spoken for by BIP86 and must not be reused for anything else.

### 4.2 Coin types

Coin types are registered externally by
[SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) and are **not**
assigned by this REP. Reddcoin's are:

| Coin type | Network | Source |
| --- | --- | --- |
| `4'` | mainnet | SLIP-0044, RDD. `nExtCoinType` in `src/chainparams.cpp` |
| `1'` | testnet, regtest | SLIP-0044, "Testnet (all coins)" |

### 4.3 Full paths in use

| Path | Used for | Source |
| --- | --- | --- |
| `m/44'/<coin>'/<account>'/<change>/<index>` | Legacy HD wallet keys | `LegacyScriptPubKeyMan::DeriveNewChildKey` |
| `m/<account>'/<change>'/<index>'` | Pre-BIP44 HD wallet keys | same function, non-BIP44 branch |
| `m/44'/0'/0'/<change>/<index>` | Descriptor wallet, legacy addresses, **mainnet** | `SetupDescriptorScriptPubKeyMans` |
| `m/49'/0'/0'/<change>/<index>` | Descriptor wallet, P2SH-SegWit, mainnet | same |
| `m/84'/0'/0'/<change>/<index>` | Descriptor wallet, native SegWit, mainnet | same |
| `m/1018'/<coin>'/<index>'` | Release manifest signing key | release-signing REP |

### 4.4 Recorded inconsistency: descriptor wallets derive at coin type `0'`

Compiling the table above surfaced a discrepancy, recorded here as found.

The legacy HD path uses `Params().ExtCoinType()`, which is `4'` on mainnet. Descriptor wallets
do not: they append `/0'` on mainnet and `/1'` on test chains, inherited unchanged from Bitcoin
Core where `0'` is the correct coin type. A Reddcoin mainnet descriptor wallet therefore derives
at `m/44'/0'/...` while a legacy HD wallet of the same seed derives at `m/44'/4'/...`.

This REP does not propose changing it. The observation is recorded because that is what a
registry is for, and because the two facts are only visibly in tension when written next to each
other. Any change would alter where existing descriptor wallets find their keys and needs its
own proposal, migration path, and compatibility analysis.

Tracked as **RED-114**. Until that is resolved, the descriptor rows in §4.3 are correct as
written: they describe what the software does, not what it should do.

### 4.5 Registry entries

The table in §4.1 lists values. This section holds the full record for each value Reddcoin
allocates **itself**, carrying the fields §5.1 requires. Values defined by a BIP or a SLIP are
not recorded here, because their defining document already is the record.

A new self-allocated value appends a block in this format.

#### `1018'` - release manifest signing

| Field | Value |
| --- | --- |
| Purpose | `1018'` |
| Path template | `m/1018'/<coin_type>'/<index>'` |
| In use | `m/1018'/4'/0'` (mainnet, first key) |
| Hardened | All elements |
| Scheme | BIP340 Schnorr signature over a release manifest, verified by the client against a compiled-in public key |
| Specified by | Release-signing REP |
| Allocation | Self-allocated, per §5.2 |
| Assigned | 2026-09-03 |
| Checked against | BIP-allocated purposes (44, 45, 47, 48, 49, 84, 86, 87); the SLIP-reserved range 10001-19999 and the SLIP index; known unregistered users (lnd `1017'`, Cardano `1852'`, Solana `501'`) |
| Result of that check | No claimant found, 2026-09-03. Not a reservation; see §6 |
| Self-allocation justified because | Keys derive from a seed dedicated to release signing that derives nothing else. No external wallet, hardware device, or third-party tool derives this path |
| Supersedes | `m/1017'/4'/0'`, withdrawn before any production key existed. `1017'` is lnd's |
| Index semantics | Key generation, from `0'`. A rotated key is the next index from the same seed, not a new seed |
| Status | Draft. No production key generated at this path yet |

The final row matters more than it looks. Every value in this entry is still free to change
while it reads Draft. Once a production key exists and a client ships pinning its public key,
the entry is a description of an unchangeable fact rather than a decision that can be revisited
(§5.3).

## 5. Assignment procedure

### 5.1 Before proposing a new purpose value

A REP proposing a new scheme MUST, before selecting a value:

1. Check the BIP-allocated purposes (44, 45, 47, 48, 49, 84, 86, 87 at time of writing) and any
   added since.
2. Check the SLIP-reserved range, 10001 to 19999, and the SLIP index.
3. Search for unregistered users of the candidate value. `1017'` would have been caught by this
   step and was not, because the step did not exist.
4. Check §4.1 of this document.
5. Record the result of that search, including the date, since absence of a claimant is a
   statement about a moment in time. A self-allocated value is recorded as a registry entry in
   §4.5; a value taken from a BIP or a SLIP needs no entry, since its defining document is
   already the record.

Every element of a new path MUST use hardened derivation unless the scheme has a specific
reason not to, stated in its REP. A non-hardened element combined with a disclosed chain code
exposes sibling keys.

### 5.2 Self-allocation versus going upstream

The test is whether anything outside Reddcoin's own tooling will ever derive the path.

**Self-allocation is acceptable** when the keys are derived only by Reddcoin software from a
dedicated seed, and no external wallet, hardware device, or third-party tool is expected to
reach them. The release signing key (`1018'`) is the clear case: it comes from a seed that
derives nothing else, and no wallet will ever display that path.

**A scheme expecting external derivation SHOULD pursue a SLIP** in the reserved 10001 to 19999
range rather than self-allocate. If a hardware wallet or third-party client is ever expected to
derive ReddID identity keys, that scheme belongs in this category, and the SLIP process should
be started early because it depends on a third party's timeline.

This distinction is worth making before the fact rather than after: a self-allocated value is
cheap to change while it exists only in a draft, and impossible to change once keys derived
from it hold something people care about.

### 5.3 Changing an entry

An entry in §4 MAY be changed only while no production key exists at that path and no released
software derives it. Once either is true the value is fixed, and the cost of changing it is
borne by every user whose keys are already there.

The purpose value for release signing was changed from `1017'` to `1018'` under exactly this
rule: no production key had been generated, so the change cost nothing. Had it been noticed one
step later it would have cost a client release.

## 6. The standing of the values recorded here

This section exists to prevent this document being read as more than it is.

**This REP does not reserve anything.** There is no authority that could grant Reddcoin a
purpose value outside the two BIP43 routes in §3.1, and this document is not one. `1018'` is
recorded here as the value Reddcoin uses. Another project may already use it, or may take it
tomorrow, and nothing here prevents that.

**Why that is nevertheless acceptable.** Purpose collisions are harmful when two schemes derive
from the same seed and produce overlapping key spaces. Reddcoin's self-allocated paths are
derived from dedicated seeds that no other software touches, so an external collision on the
number is a documentation problem rather than a key-safety problem. Where that assumption
stops holding, §5.2 requires the upstream route instead.

**What this REP does guarantee**, because it is entirely within Reddcoin's control:

- Reddcoin schemes will not collide with each other, since §5.1 requires checking §4.
- Every value in use is written down with its reasoning.
- A future maintainer can tell which values were chosen deliberately and which were inherited.

## 7. Backwards compatibility

None required. This REP is informational and records existing behaviour. No software changes,
and no existing key moves.

## 8. References

- [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) - Hierarchical
  Deterministic Wallets.
- [BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) - Purpose Field for
  Deterministic Wallets. Quoted in §3.
- [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki),
  [BIP49](https://github.com/bitcoin/bips/blob/master/bip-0049.mediawiki),
  [BIP84](https://github.com/bitcoin/bips/blob/master/bip-0084.mediawiki) - the wallet schemes
  in §4.1.
- [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) - registered coin
  types. Reddcoin is 4.
- [SLIPs index](https://github.com/satoshilabs/slips) - the process referenced by §5.2.
- [lnd `keychain/derivation.go`](https://github.com/lightningnetwork/lnd/blob/master/keychain/derivation.go)
  - the existing user of `1017'` (§2.1).
- `src/wallet/scriptpubkeyman.cpp` - `DeriveNewChildKey` and
  `SetupDescriptorScriptPubKeyMans`, the sources for §4.3 and §4.4.
- `src/chainparams.cpp` - `nExtCoinType`.

## 9. Copyright

The original material in this document is licensed under the MIT license. The passage quoted in
§3 is excerpted from BIP43 and is not covered by that grant; see the note in §3.
