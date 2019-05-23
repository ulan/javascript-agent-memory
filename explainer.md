# JavaScript agent memory API

## tl;dr

We propose adding a `measureMemory` method to the performance API that estimates the amount of memory used by JavaScript objects of the current [JavaScript agent](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-formalism).
The proposed API improves upon the existing non-standard `performance.memory` API in the following ways:

- **better security and privacy**: only objects that the current JavaScript agent can access are accounted. No size information leaks from foreign origin contexts and resources;
- **promise-based interface**: it allows the implementation to do more work on demand without janking the web page. No overhead for web pages that do not use the API;
- **stable results**: other JavaScript agents that happen to share the same heap due to implementation details of the browser do not affect the results;
- optional support for **per-frame memory** breakdown of the result;

The proposed API is limited to JavaScript memory, but it can be extended to other memory (DOM, GPU, process) retained by the JavaScript agent in future by adding new fields to the result.

## Problem

As shown in [this collection of use cases](https://docs.google.com/document/d/1u21oa3-R1FhHgrPsh8-mpb8dIFVj60wcFiM5FFrfIQA/edit#heading=h.6si74uwp7sq8) there is a need for an API that measures memory footprint of web pages in production.
The use cases include a) analysis of correlation between memory usage and user metrics, b) detection of memory regressions, c) evaluation of feature launches in A/B tests, d) memory optimization.
Currently web developers resort to the non-standard `performance.memory` API that [is used in 20%](https://www.chromestatus.com/metrics/feature/timeline/popularity/884) of page loads in Chrome.

## Related Work

### Process memory API

There is a proposal for [comprehensive memory measurement API](https://github.com/WICG/performance-memory/blob/master/explainer.md) that covers different types of memory: JavaScript, DOM, CSS, Web Workers spawned by the page, etc.
Effectively the API measures the memory footprint of the whole OS process.
The wide scope of the API is problematic for security because it is difficult to precisely specify what the API is allowed to measure.
The proposal [is currently blocked](https://github.com/mozilla/standards-positions/issues/85#issuecomment-426382208) by information leak of opaque resources.

In contrast to that our proposal is limited to JavaScript memory of the current [JavaScript agent](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-formalism).
Note that the two proposals are complementary and can share the same interface. For example, the interface proposed here can be extended to include a `processMemory` field once the process memory proposal is unblocked in future.

### Memory pressure API
There is a proposal for [memory pressure API](https://github.com/WICG/memory-pressure/blob/master/explainer.md) that notifies the application about system memory pressure events.
This gives the application an opportunity to change its behavior at runtime to reduce its memory usage if possible e.g. by freeing up caches and unused resources.
Our proposal has different [use cases](https://docs.google.com/document/d/1u21oa3-R1FhHgrPsh8-mpb8dIFVj60wcFiM5FFrfIQA/edit#heading=h.6si74uwp7sq8) such as collecting telemetry data and detecting regressions.
Thus the two proposals are orthogonal.

## Requirements and constraints
The existing non-standard `performance.memory` API has multiple issues that make its standardization difficult.
The main issue is that the API reports the size of the whole JavaScript heap, which makes it sensitive to the way the browser assigns JavaScript heaps to web pages.
This dependency on the implementation increases variability of the results, e.g. if unrelated web pages share the same heap.
More importantly, it introduces a channel for leaking size information between different origin web pages.
Our proposal resolves this issue by accounting only JavaScript objects that the calling context can access.
We allow the implementation to throw a `SecurityError` exception if it cannot guarantee that the result is not tainted by a foreign origin.

Another security related constraint that we set for our API is that it must not leak the size of foreign origin resources.
Specifically, opaque response data from the Fetch API must not be included in the result.

The interface of the existing `performance.memory` API is synchronous, which restricts the amount of work an implementation can do on invocation.
The implementation has to have the result readily available at any time to avoid blocking JavaScript code.
Maintaining the result may incur performance and memory overhead even for web pages that do not use the API.
We want to avoid such overhead and allow the implementation to compute the result on-demand and to fold the computation in other operations, e.g. garbage collection.
For this reason, the interface of our API is asynchronous and is based on Promises.

In addition to the total size, we want to report per-frame sizes.
This is useful for isolating memory usage of separate products embedded as iframes in larger web pages.
Note that this accounts only iframes that the calling code can synchronously access.
Since computing per-frame sizes can be expensive and not all web pages need it, we require that the API by default returns only the total size and provides an option to request per-frame sizes.
An implementation is allowed to throw a `NotSupportedError` exception if computing per-frame sizes is infeasible.

While the current proposal is limited to only to JavaScript memory, we want to allow future extensions to other types of memory such as DOM, CSS, GPU, process memory.
Such extensions are currently not possible because of security issues such as opaque response size leaks.

## Summary of the requirements and constraints

- No size information leak from foreign origin frames and resources.
- No overhead for web pages that do not use the API.
- Asynchronous interface to allow on-demand computation of the result that can be folded into other operations, e.g. garbage collection.
- An option to request per-frame breakdown of the result.
- Support for workers.

## Non-Goals

- Precise measurement of JavaScript memory. This may be computationally expensive. Implementations are allowed to return an estimate.
- Measurement of non-JavaScript memory (DOM, CSS, GPU, process memory). The API can be extended to support other types of memory in the future, but this is out of scope of this proposal. The main blocker is information leak of foreign-origin resources.
- Measurement of JavaScript memory of foreign-origin iframes. This a security issue.
- A mechanism to compare memory usage between browser vendors. The API results are not comparable between different browsers.
We can change the name to `performance.measureMemoryUASpecific` if it is critical to highlight that the results are not comparable for different browsers.
- A mechanism for measuring memory synchronously before and after specific JS execution.

## API Proposal

The API consists of a single method `performance.measureMemory` that accepts an optional argument indicating whether to include per-frame sizes or not.
[We can change the name to `performance.measureMemoryUASpecific` if it is critical to highlight that the results are not comparable for different browsers]

By default the method estimates the total size of all objects in the JavaScript heap that the current calling context can access:

```javascript
const result = await performance.measureMemory();

console.log(result);

// Console output:
{
  total: {
    jsMemoryEstimate: 240*MB,
    jsMemoryRange: [120*MB, 300*MB]
  }
}

```

We do not require the result to be precise.
The implementation should return an estimate and a range of possible values.
If the heap contains only one JavaScript agent, then the result is equal to the heap size i.e. similar to the existing `performance.memory.usedJSHeapSize`.
The same is the case when the API is invoked in a worker because each worker has its own heap.

If the result may leak information from a foreign origin, then the promise is rejected with a `SecurityError` exception:

```javascript
try {
  const result = await performance.measureMemory();
  // In this particular scenario, the next line is unreachable.
  console.log(result);
} catch (exception) {
  console.assert(exception instanceof SecurityError);
}
```


### Optional per-frame sizes

The caller can request per-frame sizes by passing a `{detailed: true}` option.
We explain this case on an example with two web pages and three iframes shown in figure below.
Let’s assume that the top-level browsing contexts `pageA.foo.com` and `pageB.foo.com` are not related, i.e. one is not an opener of another.
Thus there are two JavaScript agents consisting of five JavaScript [realms](https://tc39.github.io/ecma262/#sec-code-realms) with the total memory usage of 360MB.

![Figure 1. Two web pages with three iframes and their memory usage.](/example.png)

Invocation of the API in `frameA` returns the size estimates for the three JavaScript realms in the current agent:

```javascript
// In frameA.foo.com context:
const result = await performance.measureMemory({detailed: true});

console.log(result);

// Console output:
{
  current: {
    url: 'https://frameA.foo.com/',
    jsMemoryEstimate: 10*MB,
    jsMemoryRange: [5*MB, 185*MB]
  },
  other: [
    {
      url: 'https://pageA.foo.com/',
      jsMemoryEstimate: 200*MB,
      jsMemoryRange: [100*MB, 280*MB]
    },
    {
      url: 'https://frameC.foo.com/',
      jsMemoryEstimate: 30*MB,
      jsMemoryRange: [15*MB, 195*MB]
    }
  ],
  total: { // current + related: pageA, frameA, frameC
    jsMemoryEstimate: 240*MB,
    jsMemoryRange: [120*MB, 300*MB]
  }
}
```

In order to compute the estimate the implementation has to attribute each heap object to its realm.
For some heap objects this may be computationally expensive or even impossible (e.g. for shared objects).
The implementation is free to choose any heuristic for attributing such objects.

In the given example we assume that the implementation can infer the realm for 50% of the heap.
Then 180MB are unattributed. We know that `frameA` uses at least 5MB and at most 185MB.
The implementation may decide to estimate the size of `frameA` by distributing the unattributed 180MB proportionally to the attributed frame sizes:

```javascript
estimateFrameA = 5 + 180 * (5 / (5 + 100 + 15 + 50 + 10)) = 10
```

A worker agent has a single realm.
Thus, the “other” field of the result is empty for workers.
The current and the total fields have matching values.
For most implementation there will be no estimation error because each worker gets it own heap.

```javascript
// In a web worker:
const result = await performance.measureMemory({detailed: true});

console.log(result);

// Console output:
{
  current: {
    url: 'https://webworker.location/',
    jsMemoryEstimate: 100*MB,
    jsMemoryRange: [100*MB, 100*MB]
  },
  other: [],
  total: {
    jsMemoryEstimate: 100*MB,
    jsMemoryRange: [100*MB, 100*MB]
  }
}
```

If computing per-frame sizes is infeasible or too expensive then the implementation is allowed to throw a `NotSupportedError` exception.

```javascript
// In frameA.foo.com context:
try {
  const result = await performance.measureMemory({detailed: true});
  // In this particular scenario, the next line is unreachable.
  console.log(result);
} catch (exception) {
  console.assert(exception instanceof NotSupportedError);
}
```


## Security Considerations

An implementation of the API should account only the JavaScript objects that the calling JavaScript agent can access.
That is the objects that can be read or called from one of the realms of the agent.
Additionally, the implementation is free to account internal system objects on the JavaScript heap that are necessary for supporting the accounted JavaScript objects (e.g. backing stores of arrays, hidden classes, closure environments, code objects) as long as that does not leak foreign origin information.

If the implementation cannot guarantee that the result is not tainted with foreign origin information, then it is allowed to throw a `SecurityError` exception.
The implementation is also allowed to add noise to the result and limit the rate of result computation.

In the rest of this section we look at two potential sources of information leak and show how an implementation can address them.

**Source 1: other JavaScript agents.**
If the implementation creates a separate JavaScript heap for each JavaScript agent, then this is not an issue.
Otherwise, there are two solutions:
- throw a `SecurityError` exception if two or more different agents were ever loaded on the current heap.
Note this produces useful results for web pages that do not embed different-origin iframes.
- iterate the heap and account only the objects that are accessible from the calling agent.
Note that objects shared between agents must be reported as if they were private.
In other words, an implementation must not leak information of whether an object is shared or not.

**Source 2: [platform objects](https://heycam.github.io/webidl/#dfn-platform-object) (JavaScript objects that implement Web IDL interfaces).**
If the implementation stores the resources associated with a platform object outside the JavaScript heap, then this is not an issue.
Otherwise, the API may leak size information of opaque resources and resources that are guarded by security checks (e.g. opaque responses of Fetch API, image data of canvas elements).
If resources are allocated on the JavaScript heap, then there are two solutions:
- throw a `SecurityError` exception.
- iterate the heap and account platform objects without the resources associated with them.

## Performance Considerations

The performance of the API depends on how the information leak sources described in the previous section are handled.
If resources of platform objects are allocated on JavaScript heap, then the implementation will have to iterate the heap or throw a `SecurityError` exception.
Otherwise, we have the following cases:
- **[fast]** *the total size is requested and the heap contains only one JavaScript agent*: in this case the implementation can simply return the heap size which is usually available as a counter.
- **[fast]** *the API is invoked in a worker*: since each worker gets its own heap and consists of a single realm, both total size and detailed versions of the API will be fast.
- **[slow]** *the total size is requested and the heap contains multiple same-site JavaScript agents*: this case may require heap iteration or the implementation throws a `SecurityError` exception. This case is possible with site-isolation.
- **[slow]** *the total size is requested and the heap contains multiple different-site JavaScript agents*: this case may require heap iteration or the implementation throws a `SecurityError` exception.
- **[slow]** *per-frame sizes are requested in a window agent*: this case may require heap iteration or the implementation throws a NotSupportedError exception.

In the slow cases, it may be possible to fold the heap iteration into the next garbage collection and thus reduce the cost of the heap iteration. If it is not possible, then it is probably better to throw an exception.

## Future API Extensions

The API can be extended to other types of memory in the future by adding new fields to the result:

```javascript
current: {
  url: 'https://frameA.foo.com/',
  jsMemoryEstimate: 100*MB,
  jsMemoryRange: [100*MB, 100*MB],
  domMemoryEstimate: 20*MB,
  domMemoryRange: [10*MB, 50*MB],
  gpuMemoryEstimate: 10*MB,
  gpuMemoryRange: [5*MB, 15*MB],
},
```
Note that this is currently problematic due to opaque responses from Fetch API.