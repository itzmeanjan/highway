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

![data_structure_sender](./sc/data_structure_sender.jpg)

```js
function send(uint chainId, address app, byte[] message, address hop) {

    var sourceChainId = getChainId() // C1's built-in function
    var sourceApp = getSenderAddress() // C1's built-in function
    var targetChainId = chainId
    var targetApp = app
    var nonce = ... // figure it out

    emit Message(sourceChainId, sourceApp, hop, targetChainId, targetApp, nonce, message) // voila 🎉

}
```

---

ℹ️ We've one imaginary **OFFCHAIN** entity for picking events emitted from `Highway` running on **C1** & reliably passing it to `Highway` running on **C2**, which is nothing but **hop**. How exactly these **OFFCHAIN** entities work, is a blackbox to us, as of now. We'll work on their specification, once we're done with **ONCHAIN** entities.

For now, let's assume `hop` can be either an automated program/ real-user who will pick event log from **C1** & invoke `Highway`'s `receive` function on **C2**, after getting it signed by `Highway`'s trusted oracle. This oracle is an **OFFCHAIN** entity, which will only sign message to be passed, after verifying occurance of event on **C1**. `Highway` on **C2** only processes message, if it finds valid signature from its oracle. 

> It doesn't process same message more than once, by checking respective nonce of message.

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
var oracle = 0x<addr> // Truested party for gateway contract

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
        // signature verification didn't pass
        return
    }
    // proceed
}
```

> `serialize(...)` is same as what's used by **C2**'s oracle for serializing parts & signing message. **❗️ If not, signature verification won't work. ❗️**

It's obvious that receiving side also should keep some state for checking orderliness of messages received from sender.

Proposed data structure looks like

![data_structure_receiver](./sc/data_structure_receiver.jpg)

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
