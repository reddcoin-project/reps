# REP-XXXX - Signed release manifests for client-side verification

```
  REP: (unassigned)
  Layer: Applications
  Title: Signed release manifests for client-side verification
  Authors: CryptoGnasher <gnasher@reddcoin.com>
  Status: Draft
  Type: Specification
  Assigned: (pending)
  License: MIT
```

- **Scope:** release engineering and client software. **No consensus impact.** Nothing in this
  document affects block or transaction validation.
- **Target:** `reddcoin-project/reddcoin` `develop` and the `v4.22.9-regtest` release line.
- **Implementation status:** the signing side exists
  (`contrib/release-signing/sign-release-manifest.py`, `doc/release-process.md`). The client
  verification side does **not** exist yet, and no production key has been generated. This
  document specifies both so that the two cannot disagree once they do exist.
- **Uses:** BIP43 purpose `1018'`, recorded in the Reddcoin derivation path registry REP. This
  document specifies the scheme; the registry records the value (§4).

---

## 1. Abstract

This REP specifies how a Reddcoin release manifest is signed so that a running client can
verify it without any external tooling, key exchange, or network trust.

The release manifest `SHA256SUMS` is signed with a BIP340 Schnorr signature over a
domain-separated tagged hash. The signature is published as `SHA256SUMS.sig` alongside the
existing OpenPGP `SHA256SUMS.asc`. The signing key is derived from a BIP39 mnemonic at the
hardened path `m/1018'/4'/0'` and its private half never leaves an air-gapped machine. Its
x-only public key is compiled into the client, which is therefore able to verify a release
entirely locally.

Everything specified here is frozen once a client carrying the public key is released: the
derivation path, the hash tag, the message the signature commits to, and the encoding of the
signature file. That is the reason this is a specification and not a comment in a script.

## 2. Motivation

### 2.1 What a Reddcoin user can verify today

As of 4.22.9.4, a release publishes `SHA256SUMS` and a detached OpenPGP signature
`SHA256SUMS.asc`. A user who has GnuPG, who has imported the correct signing key, and who
knows to check the fingerprint against
[`contrib/builder-keys/keys.txt`](https://github.com/reddcoin-project/reddcoin/blob/develop/contrib/builder-keys/keys.txt)
can verify a download completely.

That path is sound and this REP does not replace it. What it cannot do is serve the client
itself. The assisted-upgrade work introduces a client that fetches a release and hands it to
the user, and a client cannot delegate verification to a person who is not present.

### 2.2 Why the client cannot use the OpenPGP signature

Verifying `SHA256SUMS.asc` in the client would mean either linking an OpenPGP implementation
into the client or shelling out to `gpgv`.

`gpgv` is absent from essentially every Windows machine, and Windows is the majority of the
GUI user base. A "shell out to `gpgv`, degrade gracefully when it is missing" design degrades
for most of the users it exists to protect, and the degraded path is exactly the unverified
path. Linking a full OpenPGP stack into the client to check one signature is a large
dependency and a large attack surface for a small job.

libsecp256k1 is already in the tree, already cross-compiled to all eight release hosts, and
already built with `--enable-module-schnorrsig`. `XOnlyPubKey::VerifySchnorr` already exists
at `src/pubkey.cpp:189`. A BIP340 signature therefore costs no new dependency and behaves
identically on every host we publish for.

The two signatures serve different verifiers and both are published:

| File | Verifier | Tooling required |
| --- | --- | --- |
| `SHA256SUMS.asc` | people, distro packagers, third parties | any OpenPGP implementation |
| `SHA256SUMS.sig` | the Reddcoin client | none, built in |

A Schnorr signature over a tagged hash is a bespoke format that nothing outside our client can
check. That is precisely why the `.asc` is kept: third-party verifiability is not something to
trade away.

### 2.3 Why a specification rather than an implementation detail

Every value in §3 becomes unfixable the moment a client that pins the public key is released.
A client in the field verifies against the tag, the path-derived key, and the encoding it was
compiled with. Changing any of them later means every deployed client rejects every subsequent
release, which is indistinguishable from an attack and is worse than not signing at all.

The derivation path in particular has already been changed once during development, from
`m/1017'` to `m/1018'`, because `1017'` is lnd's (§4). That change was free only because no
production key existed. Recording these values in a REP is what makes the next such change
cost visible before it is made rather than after.

## 3. Specification

### 3.1 Notation

Byte strings are written as lowercase hex. `||` is concatenation. `SHA256(x)` is the SHA-256
digest of `x`. Hardened BIP32 derivation is written with a trailing apostrophe. "The manifest"
means the exact byte sequence of the published `SHA256SUMS` file, including its final newline
and with no normalisation of any kind.

### 3.2 The release key

The release key MUST be derived as follows.

1. A BIP39 mnemonic is generated on an air-gapped machine. It MUST be a valid BIP39 phrase of
   12, 15, 18, 21 or 24 words from the English wordlist, and MUST be checksum-valid.
2. The BIP39 seed is `PBKDF2-HMAC-SHA512(password = NFKD(mnemonic), salt = NFKD("mnemonic" ||
   passphrase), iterations = 2048, dkLen = 64)`, as in BIP39. The passphrase MAY be empty.
3. The BIP32 master key is `HMAC-SHA512(key = "Bitcoin seed", data = seed)`, left 32 bytes the
   private key and right 32 bytes the chain code, as in BIP32. The literal string
   `"Bitcoin seed"` is BIP32's fixed constant and MUST NOT be substituted with a
   Reddcoin-specific value; doing so would derive a different key from the same words and
   would put the key beyond the reach of every standard recovery tool.
4. The signing key is derived at:

```
m/1018'/4'/0'
  │      │   └── index. The rotation slot: a replacement key is 1', from the same seed.
  │      └────── SLIP-0044 coin type 4, Reddcoin. Matches nExtCoinType in chainparams.
  └───────────── purpose. Allocated by this REP, see §4.
```

   Every element MUST be hardened. A non-hardened element combined with a disclosed chain code
   would expose sibling keys.

5. The public key is the 32-byte x-only encoding of the resulting point, as in BIP340.

The private key MUST NOT be transferred to, stored on, or generated by a networked machine.
Recovery is from the offline mnemonic backup, which MUST be treated with the discipline given
to a wallet seed.

### 3.3 Purpose `1018'`

See §4 for the allocation and the reasoning.

### 3.4 The signed message

The signature does not commit to `SHA256(manifest)`. It commits to a tagged hash, as taproot
does, so that a release signature can never be reinterpreted as a signature over anything else
made by a key on the same curve.

Let:

```
tag  = "Reddcoin/ReleaseManifest"          (24 ASCII bytes, no NUL, no newline)
th   = SHA256(tag)
msg  = SHA256(th || th || manifest)
```

This is exactly BIP340's tagged hash construction and exactly what `TaggedHash()` at
`src/hash.h:212` computes, so a client implementation reuses the existing primitive rather
than reimplementing it.

`msg` is the 32-byte message signed under BIP340.

The tag is part of the format. Changing a single byte of it invalidates every signature every
released client will accept.

The signature commits to the manifest **as published, byte for byte**. Implementations MUST
read the manifest as binary and MUST NOT normalise line endings, trim whitespace, reorder
lines, or re-encode. It follows that if the manifest is regenerated for any reason, both
`SHA256SUMS.asc` and `SHA256SUMS.sig` MUST be regenerated and re-uploaded with it, or both
will verify as BAD against what is served.

### 3.5 The signature file

`SHA256SUMS.sig` MUST contain the 64-byte BIP340 signature encoded as 128 lowercase
hexadecimal characters, followed by a single `\n`. No armour, no headers, no comments, no
filename, no second line.

It MUST be published in the same directory as the artifacts and the manifest, that is
`https://download.reddcoin.com/bin/reddcoin-core-<version>/`, alongside `SHA256SUMS` and
`SHA256SUMS.asc`.

Verifiers SHOULD accept surrounding whitespace and MUST reject anything else, including
uppercase hex ambiguity being resolved by the parser rather than by the file. A file that does
not decode to exactly 64 bytes MUST be rejected without attempting verification.

### 3.6 Client verification (normative)

A client that verifies a release MUST perform all of the following, in order, and MUST treat
any failure as "this release cannot be verified". It MUST NOT fall back to an unverified
download, and MUST NOT present an unverified artifact as though it were verified.

1. Fetch `SHA256SUMS` and `SHA256SUMS.sig` over TLS with certificate verification enabled.
   Transport security is required but is **not** the trust anchor; see §8.2.
2. Reject the signature file if it does not decode to exactly 64 bytes. This check is
   mandatory before step 4, because `XOnlyPubKey::VerifySchnorr` asserts on a signature length
   other than 64 and would abort the process rather than return false.
3. Compute `msg` per §3.4 over the manifest bytes exactly as received.
4. Verify the signature against the compiled-in public key using BIP340 verification. Reject on
   failure.
5. Determine the artifact filename for this host and role using the mapping in
   `src/node/release_artifacts.h`.
6. Locate the manifest line whose filename field equals that artifact name exactly. There MUST
   be exactly one such line. Zero lines, or more than one, MUST be treated as failure.
7. Compare the SHA-256 of the downloaded artifact against the digest on that line, as a
   constant-time or plain byte comparison of the 32 raw bytes. Reject on mismatch.

Only after step 7 succeeds may the client present the artifact to the user as verified.

Note that steps 1 to 4 verify the manifest, and steps 5 to 7 are what make that verification
mean anything about the file the user is about to run. A client that verifies the signature and
then fails to bind the artifact to a manifest line has verified nothing useful.

### 3.7 Key rotation

The public key is compiled into the client, so rotating it requires a client release. This cost
was accepted deliberately: any automatic client-side verification requires a trust anchor, and
the alternative on the table was verifying nothing.

A replacement key MUST be derived at `m/1018'/4'/<n+1>'` from the same seed. It MUST NOT be
derived from a fresh mnemonic unless the existing seed is believed compromised, because a new
seed means a second backup to keep alive for the lifetime of every client that pins either key.

There is deliberately **no delegation chain** (no scheme in which the old key signs the new
one). Delegation is compatible with an air gap, but it solves compromise, and the threat model
here is loss (§8.4). Delegation is useless against loss precisely because it requires the key
that was lost.

During a rotation, a release MAY be signed with both keys and published as `SHA256SUMS.sig`
(new) and `SHA256SUMS.sig.<keyid>` (old). Clients pinning either key can then verify. This is
OPTIONAL and is a deployment convenience, not part of the format.

## 4. Purpose `1018'`

The value itself is recorded in the **Reddcoin derivation path registry REP**, which holds
every purpose value Reddcoin uses and the procedure for assigning a new one. That separation is
deliberate: as ReddID, tipping, and any later scheme acquire paths of their own, allocations
scattered across the REPs that happen to need them are allocations nobody can check against.

This document specifies the subtree beneath the value:

```
m/1018'/coin_type'/index'
```

where `coin_type'` is the SLIP-0044 coin type (`4'` for Reddcoin mainnet) and `index'` is the
key generation, starting at `0'`.

Two properties of the value carry over from the registry and are restated here because a reader
of this REP needs them:

**It is self-allocated, not reserved.** BIP43 offers a scheme two routes to a purpose value: a
BIP number, or the 10001 to 19999 range reserved for SLIPs. A Reddcoin-only scheme fits neither
comfortably, so `1018'` is taken and documented, as lnd, Cardano and Solana take and document
theirs. It was checked against the BIP-allocated values and against known unregistered users
and had no claimant, but absence of a claimant is not a reservation. This is acceptable here
specifically because the release key comes from a dedicated seed that derives nothing else, so
an external collision on the number would be a documentation problem and not a key-safety one.

**Why not `1017'`.** The tool originally used `m/1017'/4'/0'`. `1017'` is lnd's:
`BIP0043Purpose = 1017` in
[`keychain/derivation.go`](https://github.com/lightningnetwork/lnd/blob/master/keychain/derivation.go),
with the hierarchy `m/1017'/coinType'/keyFamily'/0/index` and `KeyFamilyMultiSig = 0`. The old
Reddcoin path was therefore a prefix that describes a Lightning multisig branch for coin type
4.

No key would ever have collided, since the release seed derives nothing else and lnd's leaves
are five levels deep with the last two unhardened. The cost was legibility: a derivation path
is read by people, and that one said something untrue about the key it identified. It was
changed while doing so was still free, and the registry's §5.1 exists so the next value is
checked before it is chosen rather than after.

## 5. Test vectors

These use BIP39 test vector 1, whose mnemonic is public and which MUST NOT be used for
anything real. They exercise BIP39 derivation, the path in §3.2, the tagged hash in §3.4 and
BIP340 signing.

BIP340 signing here uses an auxiliary randomness value of 32 zero bytes, which makes the
signature deterministic and therefore testable. Signers MAY use real auxiliary randomness in
production; verifiers MUST accept either, since BIP340 verification does not depend on it.

### 5.1 Key derivation

```
mnemonic      abandon abandon abandon abandon abandon abandon abandon abandon
              abandon abandon abandon about
passphrase    (empty)

bip39 seed    5eb00bbddcf069084889a8ab9155568165f5c453ccb85e70811aaed6f6da5fc1
              9a5ac40b389cd370d086206dec8aa6c43daea6690f20ad3d8d48b2d2ce9e38e4

path          m/1018'/4'/0'
private key   aef43bafbe8ba85affc833f9b91675a8df527d87460765c94e0ced48549d7c94
x-only pubkey 53c9cbcba0ac673c841e72aa4a430206110feba034df81c58fba3917a80f6700
```

The seed value is BIP39's published vector 1 seed, so an implementation that produces a
different seed has a BIP39 bug rather than a Reddcoin one.

### 5.2 Manifest signature

Manifest (202 bytes, two lines, each terminated by `\n`, two spaces between digest and name):

```
1f0b3ca2d0f4bd0b04ef3f0e8a3ab88a1e1b0f7a8e1d5a3b6c9d2e4f0a1b2c3d  reddcoin-4.22.9.4-x86_64-linux-gnu.tar.gz
9e8d7c6b5a4938271605f4e3d2c1b0a9f8e7d6c5b4a39281706f5e4d3c2b1a09  reddcoin-4.22.9.4-win64.zip
```

```
sha256(manifest)  4d363d6853b51e5734c41b13bbf6bcb16c16d8c437e1d96c270dde3a02fd080a
tag               Reddcoin/ReleaseManifest
msg (tagged hash) f0b197e423efe74b8898f0a7d3e51404c20beb817caf2f86de5182741dcaf480
signature         d552158b18f0f68ee2c67404b70475baac9f324aaa7870c1735f6a1e7c146b03
                  28f7f000651917afab0e8393841985c3a1013f527b1e17a4dfe2c0d2f14f4608
```

Note that `sha256(manifest)` is listed only to help debug an implementation that has gone
wrong. It is **not** what is signed; `msg` is. An implementation that signs `sha256(manifest)`
directly will produce a signature that verifies against itself and against no released client.

## 6. Reference implementation

`contrib/release-signing/sign-release-manifest.py` implements §3.2, §3.4 and §3.5 and is the
tool the release process uses. It depends on nothing outside the Python standard library and
the Reddcoin repository, so it runs on an air-gapped machine with no package installation.

- `selftest` runs the official BIP340 vectors. It SHOULD be run on any machine before that
  machine signs anything users will trust.
- `pubkey` prints the x-only public key for a mnemonic.
- `sign` signs a manifest, and refuses to write a signature it cannot itself verify.
- `verify` checks a signature against a manifest and a public key, exiting non-zero on
  failure so an upload can be gated on it.

The tool validates the mnemonic against the BIP39 English wordlist and its checksum before
deriving. The wordlist is parsed from `src/util/lang/bip39_english.h`, the header the client
already ships, so there is a single wordlist in the repository. A mistyped word otherwise
derives a perfectly usable key for a wallet that has never signed anything, which matters most
during recovery onto a rebuilt machine.

Checksum validation does not establish that a valid phrase is the *right* phrase. The operator
MUST still compare the printed public key against the value recorded in `doc/release-process.md`
before publishing a signature.

## 7. Proposed client code

Not yet implemented. Sketch, for `src/node/` alongside the existing update-check code:

```cpp
//! The release signing key, x-only BIP340, per REP-XXXX. Compiled in: this is
//! the trust anchor, and it is deliberately not fetched from anywhere.
constexpr std::array<uint8_t, 32> RELEASE_PUBKEY{/* filled in at key generation */};

static const CHashWriter HASHER_RELEASE_MANIFEST = TaggedHash("Reddcoin/ReleaseManifest");

bool VerifyReleaseManifest(Span<const unsigned char> manifest,
                           Span<const unsigned char> sig)
{
    // Mandatory: VerifySchnorr asserts on any other length and would abort.
    if (sig.size() != 64) return false;

    CHashWriter hasher{HASHER_RELEASE_MANIFEST};
    hasher.write(reinterpret_cast<const char*>(manifest.data()), manifest.size());

    const XOnlyPubKey pubkey{Span{RELEASE_PUBKEY}};
    return pubkey.VerifySchnorr(hasher.GetSHA256(), sig);
}
```

Unit tests MUST cover, at minimum: the §5.2 vector verifying; a single flipped bit in the
manifest failing; a single flipped bit in the signature failing; a signature of length 63 and
65 being rejected without reaching `VerifySchnorr`; and a signature made under a different tag
failing, which is what proves the domain separation is actually applied.

## 8. Security considerations

### 8.1 What this protects against

A user who downloads a release through a client implementing §3.6 is protected against a
substituted or modified artifact, whatever the cause: a compromised download host, a
compromised mirror, a hostile network that can present a valid certificate for the download
domain, and corruption in transit. None of those can produce an artifact whose digest appears
in a manifest signed by a key that exists only on an air-gapped machine.

### 8.2 What it does not protect against

**Denial of service.** An attacker who controls the network can withhold the manifest, the
signature, or the artifact. The client can then decline to upgrade, which is the correct
behaviour, but it cannot upgrade. Verification prevents a bad upgrade; it does not guarantee an
upgrade.

**A compromised build.** The signature attests that the release manifest is the one the
release engineer signed. It says nothing about whether the artifacts were built from the source
they claim. That property comes from the reproducible guix builds and from independent builders
attesting to matching hashes, and is out of scope here.

**A compromised air-gapped machine.** Out of scope by construction. If the signing machine is
compromised, the signing key is compromised, and §3.7 rotation is the only remedy.

**Anything before first install.** The pinned key protects upgrades of an already-installed
genuine client. A user's very first download is protected by `SHA256SUMS.asc` and the usual
out-of-band checks, not by this.

Note that TLS certificate verification in the update path, added separately, is transport
hygiene under this design rather than a trust dependency. Even a network attacker who can
forge a certificate for the download host cannot forge a signature. This is the intended
result: the security of an upgrade should not rest on the CA system.

### 8.3 Domain separation

The release key is on the same curve as transaction keys. Signing a bare `SHA256(manifest)`
would create a 32-byte value that could be meaningful in another protocol context. The tagged
hash in §3.4 is what prevents a signature produced here from being replayed as a signature over
something else, and vice versa. It is not optional and it cannot be added later.

### 8.4 Loss of the signing key

With an air gap, the realistic failure is **loss, not compromise**: a dead disk, a lost
backup, an operator who is no longer available. Losing the seed ends the ability to sign for
every client that pins the current key, and no delegation scheme can recover from it (§3.7).

The mitigation is entirely procedural, and is therefore stated normatively: the mnemonic MUST
have at least one offline backup held separately from the signing machine, and the recovery
procedure MUST be exercised, not merely documented. A backup that has never been restored is a
hypothesis.

## 9. Deployment and sequencing

The ordering matters and has one trap in it.

1. **Generate the production key** on the air-gapped machine, at the path in §3.2. Record the
   mnemonic per §8.4. Fill the public key into `doc/release-process.md`.
2. **Publish a `SHA256SUMS.sig` with the next release**, before any client reads one.
3. **Then** ship client verification (§3.6, §7) pinning that public key.

Step 2 before step 3 is not a preference. Verification code shipped against a release stream
that carries no valid signature passes all of its own unit tests and then fails for every real
user, because there is nothing to verify. The same trap has already been hit once in this
project, when release documentation described an `SHA256SUMS.asc` that was never actually being
published.

A release published before step 2 will not carry a `.sig`. Clients from step 3 will correctly
decline to verify those older releases. This is acceptable: users move forward, and the
`.asc` remains for anyone verifying an older artifact by hand.

## 10. Backwards compatibility

No consensus impact, no P2P impact, no wallet impact.

Existing clients ignore `SHA256SUMS.sig` because they do not fetch it. Existing verification
workflows built on `SHA256SUMS.asc` continue to work unchanged, and the `.asc` continues to be
published, so no third party's tooling is affected.

Releases made before this REP is deployed carry no `.sig` and cannot retroactively be given
one that any client would benefit from, since verification and publication must be introduced
in the order given in §9.

## 11. References

- [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) - Hierarchical
  Deterministic Wallets. Source of the `"Bitcoin seed"` constant in §3.2.
- [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) - Mnemonic code for
  generating deterministic keys.
- [BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) - Purpose field for
  deterministic wallets.
- [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) - Schnorr signatures
  for secp256k1. Source of the tagged hash construction in §3.4.
- [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) - Registered coin
  types. Reddcoin is 4.
- [lnd `keychain/derivation.go`](https://github.com/lightningnetwork/lnd/blob/master/keychain/derivation.go)
  - the existing user of purpose `1017'` (§4).
- `doc/release-process.md` - the operational procedure this REP specifies the format for.
- `contrib/release-signing/sign-release-manifest.py` - reference implementation (§6).
- `src/hash.h:212` - `TaggedHash()`.
- `src/pubkey.cpp:189` - `XOnlyPubKey::VerifySchnorr()`.
- `src/node/release_artifacts.h` - host to artifact-filename mapping used by §3.6 step 5.

## 12. Copyright

This document is licensed under the MIT license.
