# Current status

In a review, transparent proxies faced significant doubts from some TC39 delegates ([details](https://github.com/littledan/proposal-proxy-transparent/issues/3)). Because of that, **the author of this proposal has no plans to present this proposal to TC39**.


-----

# Transparent Proxies

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

In general, transparent Proxies unwrap themselves before usages of relevant internal slots that would make sense to carry through. Internal slots are not directly forwarded, however, to minimize the amount of unexpected states that existing algorithms may encounter (as these can currently assume that an object with certain internal slots will not have overridden object operations).

- Reading or writing private fields
- At the beginning of some methods in the JS standard library which access internal slots, like `RegExp.prototype.exec`, `Map.prototype.get` or `TypedArray` methods, but not others which are deliberately retargetable, like `RegExp.prototype[Symbol.split]` or `Array.prototype.map`. 
- In WebIDL, e.g., at the beginning of a [WebIDL operation](https://heycam.github.io/webidl/#dfn-create-operation-function) (after step 2.1.2.2)
- JS-level code may use `Proxy.transparent.unwrap` to unwrap itself, in case it wants to achieve the same thing. The same function could be used to check for and reject transparent Proxies (which would be analogous to the behavior of built-ins that don't specially handle transparent Proxies).

### Specification integration details

The above locations in the JavaScript and embedder specifications check for the presence of an internal slot, and throw an exception if the internal slot is not present. This is how type checks are generally specified. These locations are generally the locations that make sense to upgrade to transparent proxy unwrapping.

Some advantages of unwrapping at this point:
- Editorially, it makes sense to find each of these type checking lines and convert them to a line which does checking and unwrapping--this should avoid adding additional lines of repetitive, error-prone specification text.
- Implementation-wise, transparent Proxy unwrapping fits cleanly into the "else" case of a type check, so it should not cause extra checks in the case where transparent Proxies are not used. If we put unwrapping at some other point, it might often be optimizable such that more chec
ks are not done, but over time, it's likely that the specification will get out of sync somehow or hard to interpret, and there would be additional runtime overhead.

#### GetObjectWithSlot(obj, slot)

1. Perform ? RequireObjectCoercible(`obj`).
1. If `obj` has a `slot` internal slot, return `obj`.
1. If `obj` is not a transparent Proxy, throw a `TypeError`.
1. Let `target` be the Proxy target of `obj`.
1. If `target` has a `slot` internal slot, return `target`.
1. Throw a `TypeError`.


### Object capability analysis

`Proxy.transparent` is equivalent to the existing ES2015 Proxy mechanism, plus a WeakMap mapping opted-in Proxies to targets. As such, it does not lead to any new information leak. It is proposed to be part of the JS standard library so that various built-in mechanisms, such as private field access, can access this mapping.
