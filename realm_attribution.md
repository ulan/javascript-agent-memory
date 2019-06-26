# Attributing objects to realms

Realms of the same origin can synchronously script with each other and can pass objects to each other.
Thus attributing a object to a realm for computing per-frame sizes may be ambiguous in some cases.

There are three possible ways to define the owning realm for an object:

- **Retaining realm** : if removing a realm would make an object unreachable, then the realm owns the object. This is the most precise definition. However it is computationally too expensive to do online as it requires constructing the dominator tree of the object graph.
- **Running realm**: the realm of the [running execution context](https://www.ecma-international.org/ecma-262/10.0/index.html#running-execution-context) that created the object owns the object.
- **Constructor realm**: the realm of the constructor function of the object owns the object (if the object has a constructor function).

It is possible that all three definition disagree for the same object. Here is an example:

```javascript
frameA/realmA:
  function run() {
    const frameB = document.getElementById('frameB');
    const frameC = document.getElementById('frameC');
    const object = frameB.contentWindow.constructObject();
    frameC.contentWindow.retainObject(object);
  }

frameB/realmB:
  class Foo { constructor(x) { this.x = x; } };
  function constructObject() {
    return new Foo(10);
  }

frameC/realmC:
  let retainer = null;
  function retainObject(object) {
    retainer = object;
  }
```

In this example, A is the running realm, B is the constructor realm, and C is the retaining realm.

We allow an implementation to use any of these definitions to attribute objects to realms for per-frame size computation.
Note that this ambiguity does not introduce a security issue because all realms are of the same origin.