# Level 0: Hello Ethernaut

This is the level 0 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Hack
I assume you've read instructions of the level 0 and acquired test ether. The level contract methods are injected right into our browser console, so we can start calling methods.

Let's start.
The instructions asks to call this method in console:

```javascript
await contract.info()

// Output: 'You will find what you need in info1().'
```

We follow the output and call:
```javascript
await contract.info1()

// Output: 'Try info2(), but with "hello" as a parameter.'
```

Following...
```javascript
await contract.info2("hello")

// Output: 'The property infoNum holds the number of the next info method to call.'
```

To get `infoNum` we call it's public getter:
```javascript
await contract.infoNum().then(v => v.toString())

// Output: 42
```
Note that `infoNum()` returns a BigNumber object so we convert it to string to see actual number.

Proceeding with `info42`:
```javascript
await contract.info42()

// Output: 'theMethodName is the name of the next method.'
```

Again, following output:
```javascript
await contract.theMethodName()

// Output: 'The method name is method7123949.'
```

And so:
```javascript
await contract.method7123949()

// Output: 'If you know the password, submit it to authenticate().'
```

Ok...but what's the password? Let's inspect the contracts ABI:
```javascript
contract

// Output: { abi: ..., address: ..., ...., password: f () }
```

Upon inspecting we see there's a `password` method for contract. Let's call it:

```javascript
await contract.password()

// Output: 'ethernaut0`
```

We got the password. Finally:
```javascript
await contract.authenticate('ethernaut0')
```
and confirm the transaction on metamask. Submit instance.

Level completed!