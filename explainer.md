# JavaScript agent memory API

## tl;dr

We propose adding a `measureMemory` method to the performance API that estimates the amount of memory used by JavaScript objects of the current [JavaScript agent](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-formalism).
The proposed API improves upon the existing non-standard `performance.memory` API in the following ways:

- **better security and privacy**: only objects that are accessible by the calling context are accounted. No size information leaks from foreign origin contexts and resources;
- **promise-based interface**: on-demand computation of the result with zero overhead for web pages that do not use the API;
- **stable results**: other JavaScript agents that happen to share the same heap due to implementation details of the browser do not affect the results.
- optional support for **per-frame memory** breakdown of the result;

The proposed API is limited to JavaScript memory due to security reasons, but it can be extended to other memory (DOM, GPU, process) retained by the JavaScript agent in future by adding new fields to the result once the security issues are resolved.

## Problem

As shown in [this collection of use cases](https://docs.google.com/document/d/1u21oa3-R1FhHgrPsh8-mpb8dIFVj60wcFiM5FFrfIQA/edit#heading=h.6si74uwp7sq8) there is a need for an API that measures memory footprint of web pages in production.
The use cases include a) analysis of correlation between memory usage and user metrics, b) detection of memory regressions, c) evaluation of feature launches in A/B tests, d) memory optimization.
Currently web developers resort to the non-standard `performance.memory` API that [is used in 20%](https://www.chromestatus.com/metrics/feature/timeline/popularity/884) of page loads in Chrome.
The existing API has multiple issues that make its standardization difficult.

The main issue is that the API reports the size of the whole JavaScript heap, which makes it sensitive to the way the browser assigns JavaScript heaps to web pages.
This dependency on the implementation increases variability of the results, e.g. if unrelated web pages share the same heap.
More importantly, it introduces a channel for leaking size information between different origin web pages.
Our proposal resolves this issue by accounting only JavaScript objects that the calling context can access.
We allow the implementation to throw a `SecurityError` exception if it cannot guarantee that the result is not tainted by a foreign origin.

Another security related constraint that we set for our API is that it must not leak the size of foreign origin resources.
Specifically, opaque response data from the Fetch API must not be included in the result.
Note that this issue [is currently blocking](https://github.com/mozilla/standards-positions/issues/85#issuecomment-426382208) another [proposal](https://github.com/WICG/performance-memory/blob/master/explainer.md) for a memory measurement API that reports the process memory.
What is different in our proposal is that it accounts only the JavaScript memory instead of the process memory.
We expect that most implementations do not allocate opaque response data on the JavaScript heap (because the body field of an opaque response [is set to null](https://fetch.spec.whatwg.org/#concept-filtered-response-opaque-redirect) by the spec).
So for most implementation this is not an issue. Other implementations are required to throw a `SecurityError` exception if opaque response data is present on the JavaScript heap.

The interface of the existing `performance.memory` API is synchronous.
This means that the implementation has to have the result readily available at any time to avoid blocking JavaScript code.
Maintaining the result may incur performance and memory overhead even for web pages that do not use the API.
We want to avoid such overhead and allow the implementation to compute the result on-demand and to fold the computation in other operations, e.g. garbage collection.
For this reason, the interface of our API is asynchronous and is based on promises.

In addition to the total size, we want to report per-frame sizes.
This is useful for isolating memory usage of separate products embedded as iframes in larger web pages.
Note that this accounts only iframes that the calling code can synchronously access.
Since computing per-frame sizes can be expensive and not all web pages need it, we require that the API by default returns only the total size and provides an option to request per-frame sizes.
An implementation is allowed to throw a `NotSupportedError` exception if computing per-frame sizes is infeasible.

While the current proposal is limited to only to JavaScript memory, we want to allow future extensions to other types of memory such as DOM, CSS, GPU, process memory.
For this reason, the API has a generic name: `measureMemory` instead of `measureJavaScriptMemory`.

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
If the heap contains only one JavaScript agent, then the result is precise and is equal to the heap size (i.e. equivalent to the existing `performance.memory.usedJSHeapSize`).
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

As was mentioned before, the API requires that an implementation accounts only objects that the calling context can access and does not leak information about foreign origin objects.
An implementation is free to account internal system objects (e.g. hidden classes, closure environments) on the heap as long as that does not leak cross-origin size information.
In this section we look at three potential sources of information leak and show how an implementation can address them.

**Source 1: foreign-origin browsing contexts.**
If an implementation maps different-origin agents to different JavaScript heaps, then this is not an issue.
Otherwise, there are two solutions:
- throw a `SecurityError` exception if two or more different origins were ever loaded on the heap.
Note this produces useful results for web pages that do not embed different-origin frames.
- iterate the heap and account only the objects that are accessible from the calling context.
Note that objects shared between agents must be reported as if they were private. In other words, an implementation must not leak information of whether an object is shared or not.

**Source 2: opaque responses from Fetch API.**
If the fetched data is not allocated on the JavaScript heap for opaque responses, then this is not an issue.
This is likely the case for most implementations because the spec [sets](https://fetch.spec.whatwg.org/#concept-filtered-response-opaque-redirect) the body field of an opaque response to `null`.
Otherwise, there are two solutions:
- throw a `SecurityError` exception if there was a fetch request with opaque response tainting.
- iterate the heap and account only the objects that are accessible from the calling context.
Note that the data of an opaque resource is excluded from the result because the calling context cannot access it


**Source 3: other opaque resources.**
If an implementation allocates other opaque resources on the JavaScript heap (e.g. image data, media data, cookies), then they should be handled similar to Source 2.

## Performance Considerations

Depending on the JavaScript heap organization the API implementation can be fast or slow:
- **[fast]** *the total size is requested and the heap contains only one JavaScript agent*: in this case the implementation can simply return the heap size which is usually available as a counter.
- **[fast]** *the API is invoked in a worker*: since each worker gets its own heap and consists of a single realm, both total size and detailed versions of the API will be fast.
- **[slow]** *the total size is requested and the heap contains multiple same-site JavaScript agents*: this case may require heap iteration or the implementation throws a SecurityError exception. This case is possible with site-isolation.
- **[slow]** *the total size is requested and the heap contains multiple different-site JavaScript agents*: this case may require heap iteration or the implementation throws a SecurityError exception.
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

## Related Work

- There is a [proposal](https://github.com/WICG/performance-memory/blob/master/explainer.md) to add a `performance.getMemoryEstimateUASpecific` API that measures process memory retained by the current JavaScript agent.
The memory measurement is comprehensive and intended to account all resources: DOM nodes, graphics, web worker memory, etc.
The authors list per-frame memory accounting as an explicit non-goal.
The proposal is [is currently blocked](https://github.com/mozilla/standards-positions/issues/85#issuecomment-426382208) by information leak of opaque resources.
Note that our proposal and this proposal are complementary and can reuse the same interface: e.g. the process memory can be returned as a `processMemoryEstimate` field of the result.

- The [precise MemoryMeasurement feature](https://www.chromestatus.com/feature/5128919925653504) lifts quantization and delay restrictions for processes locked to a site.
