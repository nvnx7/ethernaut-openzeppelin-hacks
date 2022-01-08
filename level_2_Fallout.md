# Level 2: Fallout


This is the level 2 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites:
- Solidity smart contract [constructors](https://docs.soliditylang.org/en/v0.8.10/contracts.html)

## Hack
Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

The `player` has to claim ownership of the contract.

Inspecting all the methods, it can be seen there isn't any method that switches the ownership of the contract. Only ownership logic is inside the constructor. But, constructors are called only once at the deployment time!

How? Something about constructor declaration looks unusual -- it is defined with same name as the contract i.e. `Fallout`. Don't we use `constructor` keyword to declare constructors?

Actually, older versions of Solidity had this kind of constructor declaration. Check the version of this contract - `v0.6.0`. If we go through the [docs](https://docs.soliditylang.org/en/v0.8.10/050-breaking-changes.html#constructors) to find out when `constructor` was favored against contract name as constructor, we find that `v0.5.0` mandated that - _"Constructors must now be defined using the constructor keyword."_

This contract defines `v0.6.0` as compiler version but mistakenly uses old, long deprecated & an invalid constructor declaration!!

That means, it'll be treated as a normal method!

Aha! Gotcha! Let's call it.

```javascript
await contract.Fallout()
```

And `player` has taken over as `owner`. Verify by:
```javascript
await contract.owner() === player

// Output: true
```

Submit instance. Done.