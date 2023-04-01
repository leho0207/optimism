# Bond Manager Interface

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Overview](#overview)
- [The Bond Problem](#the-bond-problem)
  - [Dumb Bond](#dumb-bond)
  - [Gassy Bond](#gassy-bond)
- [Types of Bond Managers](#types-of-bond-managers)
  - [OracleBondManager](#oraclebondmanager)
  - [AttestationBondManager](#attestationbondmanager)
  - [DisputeGameBondManager](#disputegamebondmanager)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

A bond manager is an abstract entity that handles bonds. This is used in a couple
of places, but most notably revolving around [propoals](./proposals.md). That is,
when outputs are able to be posted to the `L2OutputOracle` permissionlessly, there
must be some value, or bond, that the proposer posts in order to *disincentivize*
actors from posting spam and invalid outputs. These bonds, implemented as
payments in ether, are handled by this abstract bond manager.

When permissionless output proposals are enabled, the `L2OutputOracle` will need
one bond manager implementation to handle bonds being posted on proposals and to
call or retract bonds when the output is either deleted or finalized.

Additionally, when [dispute-games](./dispute-game-interface.md) are supported as
a method of challenging the output [proposals](./proposals.md), a different bond
manager will be needed since `DisputeGame`s will require more complex bond posting
including bond escalation, resolving bonds, etc, etc.

## The Bond Problem

While this seems simple, deriving a "correct" and "sound" price for a bond can be
done in a variety of different ways, each with its own tradeoffs. The two that
are used are the "Dumb Bond" and the "Gassy Bond".

### Dumb Bond

The Dumb Bond is a very conservative approach to establishing a **fixed** bond
price using the worst case gas cost for deleting an output proposal. The idea here
is that a bond that's posted for a given output proposal must at least cover the
cost of the challenge agents deleting that output proposal in order to make
honest output disputes incentive compatible.

With this approach, the Dumb Bond is fixed to `1 ether`. Working backwards, if
the Dumb Bond is `1 ether`, the cost of challenging an output proposal must not
exceed `1 ether`. This would imply a base fee of `10,000` for a challenge costing
`100,000` gas with a `1 gwei priority fee`. This is an unprecedented base gas fee
by an order of magnitude, and ought to be sufficient.

### Gassy Bond

Better bond heuristics can be used to establish a bond price that accounts for
the time-weighted gas price. Gassy Bonds use a separate oracle contract called
`GasPriceFluctuationTracker` that tracks gas fluctuations within a pre-determined
bounds. This replaces the ideal solution of tracking challenge costs over all L1
blocks, but provides a reasonable bounds. This contract is funded by the proposers
who post the bonds.

## Types of Bond Managers

Below we outline the various bond managers.

### OracleBondManager

The oracle bond manager is handles bond management on behalf of the `L2OutputOracle`.

In the `L2OutputOracle`, a permissionless proposer can post an output proposal, which
becomes finalized after a pre-determined interval detailed in the
[proposals doc](./proposals.md). When proposing an output, the proposer must
post a bond that is used to disincentivize spam. When the output is finalized,
the bond may be claimed by the proposer. But in the case that the output is invalid,
the bond *should* be seized by a set of challenge agents or a
[dispute game](./dispute-game-interface.md).

### AttestationBondManager

The attestation bond manager is the simplest bond manager; there is none!

The attestors are a set of permissioned addresses that, when a quorum is reached,
are able to delete an output proposal. Since the attestors are permissioned, they
don't need to post bonds, but rather, can only seize the bond posted to the
`L2OutputOracle` (and by extension, the [OracleBondManager](#-oraclebondmanager))
by the proposer.

### DisputeGameBondManager

When the attestations are replaced with [dispute games](./dispute-game-interface.md),
permissionless output proposals will be disputed by a `DisputeGame` instead of
the set of permissioned attestors. In these games, there will be bonds posted
at each step of the game. At the end of the game, the bonds are dispersed to the
winning "side" of the game - the challengers or defeners (who including the proposer).
The `DisputeGameBondManager` handles the management of all these bonds.
