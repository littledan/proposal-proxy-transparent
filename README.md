# Transparent Proxies

Daniel Ehrenberg

JavaScript developers sometimes use Proxies in a membrane-style information-hiding pattern, but often they just want to intercept all object operations, *including* in methods on the object or defined in subclasses. There are a couple of interactions with the rest of the language or proposals where this doesn't quite work out with current Proxy. These features don't "tunnel" through Proxy. Unless the Proxy implements a membrane pattern (which would remove the Proxy traps from references inside the class), the receiver (which is the Proxy) will be "wrong" and not have the private fields that the underlying target does.

- Private fields ([Issue](https://github.com/tc39/proposal-class-fields/issues/106))
- Internal slots ([Issue](https://github.com/tc39/ecma262/issues/1114))

This repository proposes **transparent Proxies** as a solution to the problem. Transparent Proxies are a mechanism for allowing the object model operations to be intercepted, but to permit other things (private fields and some internal slots) to pass right through to the target. Unlike ordinary Proxies, they are not an encapsulation mechanism.

## API

`Proxy.transparent(target, handlers)`

Analogous to `new Proxy`: Create a transparent Proxy with the given target and handlers. Returns a single Proxy object. Creating a transparent Proxy of a transparent Proxy throws a `TypeError`.

`Proxy.transparent.unwrap(obj)`

- If `obj` is a transparent Proxy, return the Proxy's target
- Otherwise, return `obj`

## Semantics

For the most part, transparent Proxies act exactly like ordinary Proxies. The only difference is that, in several algorithms, there is an extra step to "tunnel" down to the underlying receiver, with the `Proxy.transparent.unwrap` algorithm.

### Usage cases

In general, transparent Proxies unwrap themselves before usages of relevant internal slots that would make sense to carry through. Internal slots are not directly forwarded, however, to minimize the ???.

- Reading or writing private fields
- At the beginning of some methods in the JS standard library which access internal slots, like `RegExp.prototype.exec`, `Map.prototype.get` or `TypedArray` methods, but not others which are deliberately retargetable, like `RegExp.prototype[Symbol.split]` or `Array.prototype.map`. 
- In WebIDL, e.g., at the beginning of a [WebIDL operation](https://heycam.github.io/webidl/#dfn-create-operation-function) (after step 2.1.2.2)
- JS-level code may use `Proxy.transparent.unwrap` to unwrap itself, in case it wants to achieve the same thing.

### Object capability analysis

`Proxy.transparent` is equivalent to the existing ES2015 Proxy mechanism, plus a WeakMap mapping opted-in Proxies to targets. As such, it does not lead to any new information leak. It is proposed to be part of the JS standard library so that various built-in mechanisms, such as private field access, can access this mapping.
