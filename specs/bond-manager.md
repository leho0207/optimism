# Bond Manager Interface

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Overview](#overview)
- [The Bond Problem](#the-bond-problem)
  - [Simple Bond](#simple-bond)
  - [Variable Bond](#variable-bond)
- [Types of Bond Managers](#types-of-bond-managers)
  - [OracleBondManager](#oraclebondmanager)
  - [AttestationBondManager](#attestationbondmanager)
  - [FaultBondManager](#faultbondmanager)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The bond manager contracts handle bonds for both the [dispute-games](./dispute-game.md)
and the `L2OutputOracle` contract (detailed in [propoals](./proposals.md)).

When outputs are permissionlessly posted to the `L2OutputOracle`, the proposer must
submit some ether as a "bond" to disincentivize spam and invalid outputs. These bonds
are handled by the `L2OutputOracle`'s bond manager. The `L2OutputOracle` contract should
forward these bonds on proposals, and call bonds when outputs are either deleted or
finalized.

Similarly, the various types of [dispute-games](./dispute-game-interface.md) require
different bond management. In the simplest dispute game (the "Attestation" dispute game)
bonds will not be required since the attestors are permissioned. But in more complex games, such as the
fault dispute game, bonds will be required since the challengers and defenders perform a series of
alternating onchain transactions requiring bonds at each step. In this case, a separate bond manager than
that of the `L2OutputOracle` is required.

## The Bond Problem

At its core, the bond manager is straightforward - it escrows or holds ether and returns
it when finalized or seized. But the uncertainty of introducing bonds lies in the
bond _sizing_, i.e. how much should a bond be? Sizing bonds correctly entails a variety of
tradeoffs. Price them too high, and not enough proposers will post outputs. Price them too low, and
challengers won't be incentivized to delete outputs if the gas cost of doing so outways the bond itself.

Below, we outline two different approaches to sizing bonds and the tradeoffs of each.

### Simple Bond

The _Simple Bond_ is a very conservative approach to bond management, establishing a **fixed** bond
size using the worst case gas cost for deleting an output proposal. The idea being
that a bond posted for a given output proposal must at least cover the
cost of the challenge agents deleting that output proposal in order to make
dispute games incentive compatible.

With this approach, the Simple Bond is fixed to `1 ether`. By working backwards, we
can establish that the cost of challenging an output proposal must not
exceed `1 ether`. As such, this implies a base fee of `10,000` for a challenge costing
`100,000` gas with a `1 gwei priority fee`. This leaves challenger agents with a significant
buffer between the historical highest gas price of roughly `236` in 2020, and the base gas fee of `10,000`.

### Variable Bond

Better bond heuristics can be used to establish a bond price that accounts for
the time-weighted gas price. One instance of this called _Varable Bonds_ use a separate oracle contract,
`GasPriceFluctuationTracker`, that tracks gas fluctuations within a pre-determined
bounds. This replaces the ideal solution of tracking challenge costs over all L1
blocks, but provides a reasonable bounds. Proposers are responsible for funding this contract.

## Types of Bond Managers

Below we outline the bond manager contracts.

### OracleBondManager

The oracle bond manager is handles bond management on behalf of the `L2OutputOracle`.

In the `L2OutputOracle`, a permissionless proposer can post an output proposal, which
becomes finalized after a pre-determined interval detailed in the
[proposals doc](./proposals.md). When proposing an output, the proposer must
post a bond that is used to disincentivize spam. When the output is finalized,
the bond may be claimed by the proposer. But in the case that the output is invalid,
the bond *should* be seized by a set of challenge agents or a
[dispute game](./dispute-game.md).

### AttestationBondManager

The attestation bond manager is the simplest bond manager; there is none!

Attestors are a set of permissioned addresses that, when a quorum is reached,
are able to delete an output proposal. Since the attestors are permissioned, they
don't need to post bonds, but can only seize the proposers bond posted to the
`L2OutputOracle` (and by extension, the [OracleBondManager](#-oraclebondmanager)).

### FaultBondManager

When fault-based [dispute games](./dispute-game.md) are introduced,
permissionless output proposals can be disputed by a `FaultDisputeGame` instead of
the permissioned attestors. In these game contracts, bonds are posted
at each step of the game. Once the game is finished, bonds are dispersed to the
winners - either the output challengers or defenders (who including the proposer).
The `FaultBondManager` contract manages all these bonds.
