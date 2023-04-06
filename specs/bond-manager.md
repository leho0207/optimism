# Bond Manager Interface

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Overview](#overview)
- [The Bond Problem](#the-bond-problem)
  - [Simple Bond](#simple-bond)
  - [Variable Bond](#variable-bond)
- [Bond Manager Interface](#bond-manager-interface)
- [Bond Manager Implementation](#bond-manager-implementation)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

In the context of permissionless output proposals, bonds are a value that must
be attached to an output proposal. In this case, the bond will be paid in ether.

By requiring a bond to be posted with an output proposal, spam and invalid outputs
are disincentivized. It's important to note that it is directly disincentivized
because if the output is invalid, it will be seized by a set of challenge agents
when the output is deleted. Thus, the bond acts as a form of economic security.

Concretely, outputs will be permissionlessly proposed to the `L2OutputOracle` contract.
When calling the propose function, the ether value is sent as the bond. This bond is
then held by a bond manager contract. The bond manager contract is responsible for
both the [dispute-games](./dispute-game.md) and the `L2OutputOracle` (further detailed
in [proposals](./proposals.md)).

The bond manager will need to handle bond logic for a variety of different
[dispute-games](./dispute-game-interface.md). In the simplest "attestation" dispute game,
bonds will not be required since the attestors are permissioned. But in more complex games,
such as the fault dispute game, challengers and defenders perform a series of
alternating onchain transactions requiring bonds at each step.

## The Bond Problem

At its core, the bond manager is straightforward - it escrows or holds ether and returns
it at maturity or seized. But the uncertainty of introducing bonds lies in the
bond _sizing_, i.e. how much should a bond be? Sizing bonds correctly is a function of
the bond invariant: the bond must be greater than or equal to the cost of the next step.
If bonds are priced too low, then the bond invariant is violated and there isn't an economic
incentive to execute the next step. If bonds are priced too high, then the actors posting
bonds can be priced out.

Below, we outline two different approaches to sizing bonds and the tradeoffs of each.

### Simple Bond

The _Simple Bond_ is a very conservative approach to bond management, establishing a **fixed** bond
size. The idea behind simple bond pricing is to establish the worst case gas cost for
the next step in the dispute game.

With this approach, the Simple Bond for an Attestation dispute game can be fixed to `1 ether`.
By working backwards, we can establish that the cost of challenging an output proposal must not
exceed `1 ether`. As such, this implies a base fee of `10,000` for a challenge costing
`100,000` gas with a `1 gwei priority fee`. This leaves challenger agents with a significant
buffer between the historical highest gas price of roughly `236` in 2020, and the base gas fee of `10,000`.

### Variable Bond

Better bond heuristics can be used to establish a bond price that accounts for
the time-weighted gas price. One instance of this called _Varable Bonds_ use a
separate oracle contract, `GasPriceFluctuationTracker`, that tracks gas fluctuations
within a pre-determined bounds. This replaces the ideal solution of tracking
challenge costs over all L1 blocks, but provides a reasonable bounds. The initial
actors posting this bond are responsible for funding this contract.

## Bond Manager Interface

Below is a minimal interface for the bond manager contract.

```solidity
interface BondManager {
  /// @notice Post the bond of the bond owner.
  function postBond(address bondOwner) external;

  /// @notice Seize the bond of the bond owner.
  /// @dev `nonce` is required since the bond owner may have multiple outstanding bonds.
  function seizeBond(address bondOwner, uint256 nonce) external;

  /// @notice Claim the bond of the bond owner.
  /// @dev `nonce` is required since the bond owner may have multiple outstanding bonds.
  function claimBond(address bondOwner, uint256 nonce) external;
}
```

## Bond Manager Implementation

Initially, the bond manager will only be used by the `L2OutputOracle` contract
for output proposals in the attestation [dispute game](./dispute-game.md). Since
the attestation dispute game has a permissioned set of attestors, there are no
bonds required.

In the future however, Fault-based dispute games will be introduced and will
require bond management. In these games, bonds will be posted at each step of
the dispute game. Once the game is finished, bonds are dispersed to the
winners - either the output challengers or defenders (who including the proposer).
The `FaultBondManager` contract manages all these bonds.
