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

How? Something about constructor declaration looks unusual -- it "seems" to be defined with same name as the contract i.e. `Fallout` (it is NOT actually). Don't we use `constructor` keyword to declare constructors?

First of all - intended constructor declaration has typo, `Fal1out` instead of `Fallout`. So it is just treated as a normal method not a constructor. Hence we simply call it & claim ownership.

Secondly, even if it wasn't a typo, that is - constructor was declared as `Fallout`, it won't even compile! 
Older versions of Solidity had this kind of constructor declaration. If you go through the [docs](https://docs.soliditylang.org/en/v0.8.10/050-breaking-changes.html#constructors) you'll find that `constructor` keyword was favored against contract name as constructor. It was mandated even in `v0.5.0` - _"Constructors must now be defined using the constructor keyword"_. And target contract uses `v0.6.0`.

Anyway, silly mistake.

```javascript
await contract.Fal1out()
```

And `player` has taken over as `owner`. Verify by:
```javascript
await contract.owner() === player

// Output: true
```

Submit instance. Done.