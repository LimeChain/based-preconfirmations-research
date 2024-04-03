# Problem Statement and Goal Definition

[Based rollups design](https://ethresear.ch/t/based-rollups-superpowers-from-l1-sequencing/15016) , proposed by [Justin Drake](https://twitter.com/drakefjustin) and the EF research team, grows in popularity increasingly due to having fewer trust assumptions and the enhanced security properties coming from based sequencing. One of the major hurdles discouraging the rollup teams to adopt the approach is the lack of clear and detailed understanding of how [preconfirmations](https://ethresear.ch/t/based-preconfirmations/17353) are going to be achieved in based rollups. Preconfirmations are quick confirmation guarantees of inclusion of an L2 transaction before the enclosing L2 block is sequenced on L1. This alleviates major UX hurdles introduced by the slower pipeline of sequencing L2 blocks in L1.

In order to support preconfirmations a preconfirmations system and mechanism needs to be designed. This system needs to ensure efficiency, efficacy and security for all actors engaging in preconfirmations.

This repository is a deliverable on a research grant awarded to Limechain for research into viable preconfirmations system and mechanism.
Naturally this research also covers suggested sequencing mechanism that serves as the foundation in order for preconfirmations to be achiavable.
# Contents

- [Glossary](./docs/glossary.md) - a glossary with commonly used terms throughout the research.
- [Vanilla Based Sequencing](./docs/vanilla-based-sequencing.md) - a decentralised sequencing and exploration serving as foundation for preconfirmations.
- [Preconfirmations for Vanilla Based Rollups](./docs/preconfirmations-for-vanilla-based-rollups.md) - a high level mechanism design for preconfirmations support for vanilla based rollups.

# Authors
- [George Spasov](https://github.com/Perseverance) - Limechain
- [Daniel Ivanov](https://github.com/Daniel-K-Ivanov) - Limechain