<pre>
  SHIP: 002
  Title: Reduce the number of delegates in Z-Network
  Authors: TechnoL0g
  Status: Active
  Type: Core/Protocol
  Created: 2020-07-12
  Last Update: 2020-07-13
</pre>

Abstract
========

Reduce the number of delegates in the mainnet network to 21.

Motivation
==========

- Achieve the most efficient transactional spread between forgers.
- Improving blockchain productivity for business, games and payments.
- Delegate Rewards Improvement.

Specifications
==============

Create a soft fork on block-N, after which the number of delegates of the signing blocks will be reduced to 21.

1 block is created every 8 seconds, 10 800 blocks per day (86400 / 8 = 10800).
With 21 forgers, each creates 514 blocks per day (10800 / 21 = 514).
