# Reddcoin Enhancement Proposals (REPs)

REP stands for Reddcoin Enhancement Proposal. Similar to Bitcoin's [BIPs](https://github.com/bitcoin/bips/), a REP is a design document providing information to the Reddcoin community, or describing a new feature for Reddcoin or its processes or environment. The REP should provide a concise technical specification of the feature and a rationale for the feature.

Because Reddcoin is forked from the Bitcoin codebase, many of the BIPs can be applied to Reddcoin as well. The purpose of the REPs is not to duplicate those which exist as BIPs, but to introduce protocol upgrades or feature specifications which are unique to Reddcoin, particularly those related to:

- **Proof-of-Stake-Velocity (PoSV)** consensus mechanism
- **Social tipping** and micropayment features
- **ReddID** identity and naming system
- **Network upgrade coordination** mechanisms

## Contributions

We use the same general guidelines for introducing a new REP as specified in [BIP 2](https://github.com/bitcoin/bips/blob/master/bip-0002.mediawiki), with a few differences. Specifically:

* Instead of the BIP editor, initiate contact with the Reddcoin Core development team and your request should be routed to the REP editor(s). The REP workflow mimics the BIP workflow.
* Recommended licenses include the MIT license
* Markdown format is the preferred format for REPs
* Following a discussion, the proposal should be submitted to the REPs git repository as a pull request. This draft must be written in BIP/REP style as described in [BIP 2](https://github.com/bitcoin/bips/blob/master/bip-0002.mediawiki), and named with an alias such as "rep-johndoe-newfeature" until the editor has assigned it a REP number (authors MUST NOT self-assign REP numbers).

## Reddcoin Enhancement Proposal Summary

Number | Layer | Title | Owner | Type | Status
--- | --- | --- | --- | --- | ---
[0001](docs/rep-0001.md) | | Reddcoin Enhancement Proposal Process | Reddcoin Core | Process | Draft
[0002](docs/rep-0002.md) | Consensus | Versionbits `lockinontimeout` (BIP8 guaranteed activation) | Reddcoin Core | Specification | Draft
[0003](docs/rep-0003.md) | Consensus | PoSV3 Stake-Timestamp Hardening | Reddcoin Core | Specification | Draft
[0004](docs/rep-0004.md) | Consensus | Per-block Stake Modifier (hard fork) | Reddcoin Core | Specification | Draft
[0005](docs/rep-0005.md) | Applications | Reddcoin BIP32 derivation path registry | Reddcoin Core | Informational | Draft

## License

Unless otherwise specified, Reddcoin Enhancement Proposals (REPs) are released under the terms of the MIT license. See [LICENSE](LICENSE) for more information or see the [MIT License](https://opensource.org/licenses/MIT).
