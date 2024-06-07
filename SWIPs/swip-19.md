---
SWIP: 19
title: Neighbourhood hopping
author: György Barabás <gyorgy@ethswarm.org> (@dysordys), Viktor Trón <viktor@ethswarm.org> (@zelig), Mark Bliss <marko@ethswarm.org>
discussions-to: https://discord.gg/Q6BvSkCv
status: Draft
type: Standards Track
category: Core
created: 2024-05-08
---

## Abstract

This SWIP describes the process to support **neighbourhood hopping**, i.e., moving a node to another neighbourhood without the need to pay the stake each time.

## Objectives

Enable a node to change its overlay address while preserving the stake it already possesses.

## Context

Neighbourhood hopping is currently supported by the Bee client. The overlay address, which specifies which neighbourhood a node falls into, is derived by hashing the node key together with an arbitrary overlay nonce chosen by the user (refer to the figure below).

![](assets/swip-19/overlay-definition.png)

Bee already supports placing one's node in an arbitrary neighbourhood by mining an overlay nonce that will result in the corresponding neighbourhood.

Currently, the staking smart contract associates a stake with an overlay address. Therefore, neighbourhood hopping requires the node operator to re-stake their node each time they change neighbourhoods. This is considered unfair and contrary to the system's ethos, as the ability to change neighbourhoods fluently is essential for the balanced distribution of nodes across the network.

## Specification

To support neighbourhood hopping with transferable stakes, the staking contract should register the user's Ethereum address instead of their overlay address.

## Implementation notes

The implementation requires change of indices: from one keyed by the overlay address to another keyed by Ethereum address. The staking API already requires the sender to send the so-called overlay nonce to the nodes to calculate the overlay address. Therefore, there will be changes to the interface functions.

### Staking contract changes

#### The overlay change endpoint:

1. Set the new nonce to

   `function changeOverlay(bytes32 _nonce)`

   node address is auto picked from the msg.sender and used to calculate new overlay

2. Event will be trigered to show that overlay has been changed

   `emit OverlayChanged(msg.sender, overlay);`

#### The stake deposit endpoint:

From previous function we now have the one where we don't send nodes addrees it is auto picked from sender and calculated

`function depositStake(bytes32 _nonce, uint256 _amount)`

We also changed the event, now it emits different indexed variable which is owner of node

`emit StakeUpdated(msg.sender, updatedAmount, overlay, block.number);`

#### Withdraw from stake

Withdraw will now also be done with address and not overlay so this should be called, address is auto picked from msg.sender

`function withdrawFromStake(uint256 _amount)`

#### Freez and slash deposit

Both freeze and slash deposit are now done by address so we call

`function freezeDeposit(address _owner, uint256 _time)`
`function slashDeposit(address _owner, uint256 _amount)

and emit

`emit StakeFrozen(_owner, stakes[_owner].overlay, _time);`
`emit StakeSlashed(_owner, stakes[_owner].overlay, _amount);`

#### Additional read only function changes

From `function overlayNotFrozen(bytes32 overlay)` to `function addressNotFrozen(address _owner)`

From `function usableStakeOfOverlay(bytes32 overlay)` to `function usableStakeOfAddress(address _owner)`

From `function lastUpdatedBlockNumberOfOverlay(bytes32 overlay)` to `function lastUpdatedBlockNumberOfAddress(address _owner)`

From `function ownerOfOverlay(bytes32 overlay)` to `function overlayOfAddress(address _owner)`

### Redistribution contract changes

As in Staking contract where we removed usage of overlay sending as it is auto fetched from staking contract and msg.sender, the same principle is applied here

#### Commits

Commits are now done just with nonce and obfsHash

`function commit(bytes32 _obfuscatedHash, uint64 _roundNumber)`

#### Reveals

Reveals are also done without overlay so we just need nodes to send these parameters

`function reveal(uint8 _depth, bytes32 _hash, bytes32 _revealNonce)`

#### Other

We also now have read function with just depth parameters, address is fetched from msg.sender

`function isParticipatingInUpcomingRound(uint8 _depth)`

we also have overload function with 2 parameters

`function isParticipatingInUpcomingRound(address _owner, uint8 _depth)`

which works as before, but is left more for debugging purposes and its safe for nodes to use the first one.

### Bee changes

Since stakes are keyed by the address the signiture of the stake accessor changed this has impactimpact on the Bee's blockchain interface as used by the stakes GET endpoint.

In particular the overlay argument of the accessor needs to be changed to Ethereum address.

The way Bee supports neighborhood change only on fresh start but in order to accomodate this change we need to allow overlay mining: target neighborhood setting must work even if the Ethereum address is already staked.

## Backward compatibility

Since we are changing how information is passed (both the overlay address and the Ethereum address), this change is not backwards-compatible.

## Test cases

Case study, with individual items needing to be tested:

1. Can one stake a node?
2. Is it then possible to change neighbourhoods?
3. If so, then did that happen with the stake transferred without a change?

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
