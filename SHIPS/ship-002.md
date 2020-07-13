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

Reduce the number of delegates in the first network (Z-Network) to 27.

Motivation
==========

- Achieve the most efficient transactional spread between forgers.
- Improving blockchain productivity for business, games and payments.
- Delegate Rewards Improvement.

Specifications
==============

Create a soft fork on block-N, after which the number of delegates of the signing blocks will be reduced to 27.

1 block is created every 8 seconds, 10 800 blocks per day (86400 / 8 = 10800).
With 27 forgers, each creates 400 blocks per day (10800 / 27 = 400).
