<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Bond Manager Interface](#bond-manager-interface)
  - [Overview](#overview)
  - [The Bond Problem](#the-bond-problem)
    - [Dumb Bond](#dumb-bond)
    - [Gassy Bond](#gassy-bond)
  - [Implementing the Bond Managers](#implementing-the-bond-managers)
    - [BondManager Interface](#bondmanager-interface)
    - [DisputeGameBondManager](#disputegamebondmanager)
    - [OracleBondManager](#oraclebondmanager)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Bond Manager Interface

## Overview

A bond manager is an abstract entity that handles bonds. This is used in a couple
of places, but most notably revolving around [propoals](./proposals.md). That is,
when outputs are able to be posted to the `L2OutputOracle` permissionlessly, there
must be some value, or bond, that the proposer posts in order to _disincentivize_
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



## Implementing the Bond Managers

Below we specify the bond manager implementations used to support permissionless
output proposals.

### BondManager Interface


### DisputeGameBondManager


### OracleBondManager

