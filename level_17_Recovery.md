# Level 17: Recovery

This is the level 17 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- Basic [Etherscan](https://rinkeby.etherscan.io) inspection
- [Determining address of a new contract](https://swende.se/blog/Ethereum_quirks_and_vulns.html)

## Hack
Given contract:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  using SafeMath for uint256;
  // public variables
  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) public {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value.mul(10);
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender].sub(_amount);
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

`player` has to retrieve the funds from the lost address of contract which was created using the `Recovery`'s first transaction.

If the address of the lost `SimpleToken` address is retrieved it's funds can be retrieved using the `destroy` method.

The easiest way to solve this would be to copy the address of `Recovery` in [Etherscan](https://rinkeby.etherscan.io/) (on Rinkeby network) and inspect transactions in **Internal Txns** tab. Find the latest **Contract Creation** transaction and click through the same to get the address of created contract.

Now simply call `destroy` method at that address.
So, if `tokenAddr` is the retrieved address then:
```javascript
functionSignature = {
    name: 'destroy',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: '_to'
        }
    ]
}

params = [player]

data = web3.eth.abi.encodeFunctionCall(functionSignature, params)

await web3.eth.sendTransaction({from: player, to: tokenAddr, data})
```
That'd do the job.

Another way to get the lost address is by utilizing the fact that creation of addresses of Ethereum is deterministic and can be calculated by:
```
 keccack256(address, nonce)
```
where `address` is the address of contract that created the transaction and `nonce` is the number of contracts the creator address has created. You can read more [here](https://swende.se/blog/Ethereum_quirks_and_vulns.html).

That's it!


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
