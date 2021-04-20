# highway
Cross Blockchain Message Passing Protocol

![architecture](./sc/architecture.jpg)

## Motivation

We're living in a time when new blockchain(s) popping up every single day & we've our assets distributed across all those self-sovereign chains, each using similar/ different mechanism for reaching consensus.

This project is an attempt in creating a generic framework for passing arbitrary messages in between applications running on two different chains. These applications are distributed in nature, but their state is not necessarily shared across multiple chains. This project will attempt to standardise some interfaces so that dApps running on different chains can talk to their peers over a reliable, ordered, authentic channel.

In future date contributors can add support for more chains, following specification defined below, for reliably passing ordered, authenticated messages between a pair of chains. This enables users of this framework to distribute their application state across two or more different chains & build some interesting projects.

> One thing I'd like to clear, this project doesn't think any chain as **root chain ( i.e. L1 ) / child chain ( i.e. L2 )**, rather every participating chain is considered to be operating independently.

---

We need two components for reliably passing messages between two applications running on two different chains.

- OnChain Entity
- OffChain Entity

Let's expand on each of these.

### OnChain Entity

Let's say application **A1**, **A2** running on chain **C1**, **C2** respectively, wants to talk to each other, below is a proposed flow.

---

**A1** invokes `send(uint chainId, address app, byte[] message, address hop)` method of `Highway` dApp deployed on **C1**, as an effect of user triggered tx **T1**, which will result into emission of event log of below form

```js
Message(uint sourceChainId, address sourceApp, address hop, uint targetChainId, address targetApp, uint nonce, bytes[] message)
```

We asume, in block **B1** of chain **C1**, tx **T1** is included. How this event log will be captured by **OFFCHAIN** entity i.e. `hop`, we'll see that in sometime.

For emitting event of aforementioned form, we need to hold some state in `Highway` on **C1**.

> Think of `Highway` on **C1** as a gateway to send message to outer world, where some outer world entity i.e. `hop` will pick it up & send it to recipient chain's `Highway`, which will in turn attempt to verify it.

For each dApp, on any possible chain _( != **C1** )_, to which any application on **C1** might ever want to send message, we're keeping some state using following data structure

```js
var state = allocate(map[address]map[uint]map[address]uint)
```

> `allocate(...)` only allocates space for top level associative array, if it's a nested structure.

Ok, let's split this data structure into it's components & see what each level serves.

**Flow :**

When **A1** invoked `Highway.send(uint chainId, address app, byte[] message, address hop)`, we start by finding out if **A1** has ever attempted to send any message to outer world or not. This can be done by doing a O(1) look up

```js
var _, ok = state[A1]
if ok {
    // yes it tried
} else {
    // no, first attempt
}
```

If **NO**, we're creating following entries

```js
state[A1] = allocate(map[uint]map[address]uint)
state[A1][chainId] = allocate(map[address]uint)

var nonce = state[A1][chainId][app] // default value 0
state[A1][chainId][app]++
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
state[A1][chainId] = allocate(map[address]uint)

var nonce = state[A1][chainId][app]
state[A1][chainId][app]++
```

If **YES**, we'll lookup respective nonce

```js
var nonce = state[A1][chainId][app]
state[A1][chainId][app]++ // for next message
```

So we've finally figured out `nonce` to be used for emitting event

```js
function send(uint chainId, address app, byte[] message, address hop) {

    var sourceChainId = getChainId() // C1's built-in function
    var sourceApp = getSenderAddress() // C1's built-in function
    var targetChainId = chainId
    var targetApp = app
    var nonce = ... // we just figured it out

    emit Message(sourceChainId, sourceApp, hop, targetChainId, targetApp, nonce, message) // voila 🎉

}
```

---

ℹ️ We've one imaginary **OFFCHAIN** entity for picking events emitted from `Highway` running on **C1** & reliably passing it to `Highway` running on **C2**, which is nothing but **hop**. How exactly these **OFFCHAIN** entities work, is a blackbox to us, as of now. We'll work on their specification, once we're done with **ONCHAIN** entities.

For now, let's assume `hop` can be either an automated program/ real-user who will pick event log from **C1** & invoke `Highway`'s `receive` function on **C2**, after getting it signed by `Highway`'s trusted oracle. This oracle is an **OFFCHAIN** entity, which will only sign message to be passed, after verifying occurance of event on **C1**. `Highway` on **C2** only processes message, if it finds valid signature from its oracle.

---

Let's now step-by-step go through what happens, when `Highway` on **C2** receives message sent by **A1** of **C1**, in form

```js
function receive(uint sourceChainId, address sourceApp, address hop, uint targetChainId, address targetApp, uint nonce, byte[] message, byte[] signed) {

    // ...

}
```

First it'll check whether it's `hop` who has invoked method or not

```js
function receive(...) {
    var invoker = getSenderAddress() // C2's built-in function
    if invoker != hop {
        // only hop should do it
        return
    }
    // proceed
}
```

Now it'll verify signature, to check whether its oracle has signed received message or not. For doing so, it'll first construct message which was signed by oracle.

```js
function receive(...) {
    var message = serialize({
        sourceChainId,
        sourceApp,
        hop,
        targetChainId,
        targetApp,
        nonce,
        message
    }) // C2's built-in function for serializing object into byte array

    if getSigner(message, signed) != oracle {
        // signature match didn't pass
        return
    }
    // proceed
}
```

It's obvious that receiving side also should keep some state for checking orderliness of messages received from sender.

Proposed data structure looks like

```js
var state = allocate(map[address]map[uint]map[address]uint)
```

First we start by checking whether this message is for this chain or not

```js
var chainId = getChainId() // C2's built-in function
if chainId != targetChainId {
    // not for us
    return
}
// proceed
```

We can now try to see if this is first time **A2** is receiving message from outside world or not, which can be done via this look up

```js
var _, ok = state[A2]
if ok {
    // not first time
} else {
    // it's first time
}
```

If **YES**, we need to create some entries, before performing orderliness check

```js
state[A2] = allocate(map[uint]map[address]uint)
state[A2][sourceChainId] = allocate(map[address]uint)

var expectedNonce = state[A2][sourceChainId][sourceApp] // default value 0
if expectedNonce != nonce {
    // ordering not being respected
    return
}
// we go to next step
```

If **NO**, we'll try to find if it's first time **A2** receiving message from `sourceChainId`, which can be done by

```js
var _, ok = state[A2][sourceChainId]
if ok {
    // not first time
} else {
    // yes it's
}
```


If **YES**, we create an entry for future usage, after that we'll do our usual orderliness check

```js
state[A2][sourceChainId] = allocate(map[address]uint)

var expectedNonce = state[A2][sourceChainId][sourceApp]
if expectedNonce != nonce {
    // ordering not being respected
    return
}
// we go to next step
```

If **NO**, we directly lookup expected nonce for this combination

```js
var expectedNonce = state[A2][sourceChainId][sourceApp]
if expectedNonce != nonce {
    // ordering not being respected
    return
}

// incrementing expected nonce, for next message
state[A2][sourceChainId][sourceApp]++
```

If orderliness is tested to be passing, `Highway` will invoke `onReceive` method of **A2** running on **C2**.

```js
function onReceive(uint chainId, address app, address hop, byte[] message) {

    // ...
    // A2 ( on C2 ) can now process this message from A1 ( on C1 )
    //
    // These messages are received in ordered, authenticated, verified form

}
```

---

> **Batching, being considered, for reducing number of round trips**

> **Specification writing in progress**
