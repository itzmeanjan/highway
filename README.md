# highway
Cross Blockchain Message Passing Framework

## Motivation

We're living in a time when new blockchain(s) popping up every single day & we've our assets distributed across all those self-sovereign chains, each using different mechanism for reaching consensus.

This project is an attempt in creating a generic framework for passing arbitrary messages in between applications running on two different chains. These applications are distributed in nature, but their state is not necessarily shared across multiple chains. This project will attempt to standardise some interfaces so that in future date contributors can add support for more chains, for reliably passing ordered, authenticated messages between a pair of chains. This way users of this framework can distribute their application state on two different chains & build some interesting applications. 

> One thing I'd like to clear, this project doesn't think any chain as **root chain ( i.e. L1 )**, rather every participating chain is very independent.

We need two components for reliably passing messages between two applications running on two different chains.

- OnChain Entity
- OffChain Entity

Let's expand on each of these.

### OnChain Entity

Let's say application **A1**, **A2** running on chain **C1**, **C2** respectively, wants to talk to each other, below is a proposed flow.

---

**A1** invokes `send(uint chainId, address app, byte[] message)` method of `Highway` dApp deployed on **C1**, as an effect of user triggered tx **T1**, which will result into emission of event log of below form

```js
Message(uint sourceChainId, uint targetChainId, address targetApp, uint nonce, bytes[] message)
```

In block **B1** of chain **C1**, tx **T1** is included. How this event log will be captured by **OFFCHAIN** entity, we'll see that in sometime.

For emitting event of aforementioned form, we need to hold some state in `Highway` on **C1**.

> Think of `Highway` on **C1** as a gateway to send message to outer world, where some outer world entity will pick it up & send _( not always )_ it to recipient chain's `Highway`, which will eventually send it to `targetApp` i.e. **A2** on **C2**.

For each dApp, on any possible chain, to which any application on **C1** might ever want to send message, we're keeping state using following data structure

```js
var state = map[address]map[uint]map[address]uint
```

Ok, let's split this data structure into it's components & see what each level serves.

**Flow**

When **A1** invoked `Highway.send(uint chainId, address app, byte[] message)`, we start by finding out if **A1** has ever attempted to send any message to outer world or not. This can be done by doing a O(1) look up

```js
var _, ok = state[A1]
if ok {
    // yes it tried
} else {
    // no, first attempt
}
```

If **NO**, we're creating an entry of following kind

```js
state[A1] = map[uint]map[address]uint // allocate memory
state[A1][chainId] = map[address]uint // allocate memory
var nonce = state[A1][chainId][app]++ // it was initially 0
```

If **YES**, we'll look up if **A1** ever tried to send message to `chainId` or not, which can be done by looking up

```js
var _, ok = state[A1][chainId]
if ok {
    // yes, it did
} else {
    // no, it didn't, first attempt to `chainId`
}
```

If **NO**, we'll create an entry for `chainId`

```js
state[A1][chainId] = map[address]uint // allocated memory for `chainId`
var nonce = state[A1][chainId][app]++ // it was initially 0
```

If **YES**, we'll lookup respective nonce

```js
var nonce = state[A1][chainId][app]++ // it was initially 0
```

So we're finally figured out `nonce` to be used for emitting event

```js
var sourceChainId = getChainId() // C1's built-in method
var targetChainId = chainId
var targetApp = app
var nonce = ... // we've figured it out

emit Message(uint sourceChainId, uint targetChainId, address targetApp, uint nonce, bytes[] message) // voila 🎉
```
---


**Specification writing in progress**
