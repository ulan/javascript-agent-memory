# Generalized and extended version of this proposal is available at [https://github.com/ulan/performance-measure-memory](https://github.com/ulan/performance-measure-memory)

---

# JavaScript Memory API

Last updated: 2019.06.12

## tl;dr

We propose adding a `measureMemory` method to the performance API that estimates the amount JavaScript objects that the calling context can access.
The proposed API improves upon the existing non-standard `performance.memory` API in the following ways:

- **better security and privacy**: only objects of the current [JavaScript agent](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-formalism) that have the same origin as the calling context are accounted. No size information leaks from foreign origin contexts and resources;
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

By default the method estimates the total size of all objects on the JavaScript heap that the current calling context can access:

```javascript
const result = await performance.measureMemory();

console.log(result);

// Console output:
{
  total: {
    jsMemoryEstimate: 200*MB,
    jsMemoryRange: [100*MB, 300*MB]
  }
}

```

We do not require the result to be precise.
The implementation should return an estimate and a range of possible values.
If the heap contains a single JavaScript agent consisting of same-origin realms, then the result is equal to the heap size i.e. similar to the existing `performance.memory.usedJSHeapSize`.
The same is the case when the API is invoked in a worker because each worker has its own heap.

If there are multiple JavaScript agents or different-origin realms, then the API accounts only the objects of the same-origin realms that the current context can synchronously script with.
We illustrate that on an example with two web pages and four iframes shown in figure below.
Let’s assume that the top-level browsing contexts `a.foo.com/page1` and `a.foo.com/page2` are not related, i.e. one is not an opener of another. Thus there are two JavaScript agents consisting of six realms with total memory usage of 500MB

![Figure 1. Two web pages with four iframes and their memory usage.](/example.png)

Invoking the API in the context of `frame1` accounts only the objects of `page1` and `frame1`.
The result will be around 240MB.
Objects of `page2` and `frame4` are skipped because they belong to a different JavaScript agent.
Objects of `frame3` belong to the same JavaScript agent and have the same origin as `frame1`, but they are skipped because `frame3` is embedded in a foreign-origin frame.

The implementation must reject the promise with a `SecurityError` exception if it cannot guarantee that the result does not leak information from a foreign origin:

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
Invocation of the API in `frame1` of the previous example returns the size estimates for the same-origin objects accessible from `frame1`:

```javascript
// In frameA.foo.com context:
const result = await performance.measureMemory({detailed: true});

console.log(result);

// Console output:
{
  current: {
    url: 'https://a.foo.com/frame1',
    jsMemoryEstimate: 30*MB,
    jsMemoryRange: [20*MB, 300*MB]
  },
  other: [
    {
      url: 'https://a.foo.com/page1',
      jsMemoryEstimate: 170*MB,
      jsMemoryRange: [80*MB, 300*MB]
    }
  ],
  total: { // frame1 + page1
    jsMemoryEstimate: 100*MB,
    jsMemoryRange: [100*MB, 300*MB]
  }
}
```

Attribution of objects to frames is implementation dependent if frames pass the objects to each other.
See [realm_attribution.md](realm_attribution.md) for the discussion of this issue.

A worker agent has a single realm.
Thus, the `other` field of the result is empty for workers.
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

## Trade-offs in API Design

For any memory measurement API there are fundamental trade-offs between:
- Security,
- Completeness and accuracy of results,
- Complexity of implementation.

Relaxing the security requirements by allowing some information leaks between different origins would make implementation simpler.
For example, relaxing the security model to site-based instead of origin-based yields much simpler implementation in browsers with site isolation.
Another example is to allow information leaks and mitigate them by adding noise and delaying the results.

This proposal provides strong security guarantee of no cross-origin information leaks.
As a result this necessarilly complicates the implementation.


## Security Considerations

An implementation of the API should account only the JavaScript objects that the calling context can access.
That is the objects that can be read or called from the current realm.
Additionally, the implementation is free to account internal system objects on the JavaScript heap that are necessary for supporting the accounted JavaScript objects (e.g. backing stores of arrays, hidden classes, closure environments, code objects) as long as that does not leak foreign origin information.

If the implementation cannot guarantee that the result is not tainted with foreign origin information, then it must throw a `SecurityError` exception.

In the rest of this section we look at two potential sources of information leak and show how an implementation can address them.

**Source 1: other JavaScript agents and foreign-origin realms.**
If the implementation creates a separate JavaScript heap for each JavaScript agent and the current JavaScript agent consists of only the same-origin realms, then this is not an issue.
Otherwise, the following solutions are possible
- throw a `SecurityError` exception if two or more different origins were ever loaded on the current heap.
Note this produces useful results for web pages that do not embed different-origin iframes.
- iterate the heap and account only the objects that are accessible from the calling agent.
- keep track of realm sizes at object allocation.
- segregate objects on the heap by realms.

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
- **[fast]** *the total size is requested and the heap contains only one JavaScript agent consisting of same-origin realms *: in this case the implementation can simply return the heap size which is usually available as a counter.
- **[fast]** *the API is invoked in a worker*: since each worker gets its own heap and consists of a single realm, both total size and detailed versions of the API will be fast.
- **[slow]** *the total size is requested and the heap contains different-origin realms*: this case may require either heap iteration, or heap segregation by origin, or accounting on allocation, or throwing a `SecurityError` exception.
Realms trusting each other may opt-in to be treated as same-origin for the purposes of this API via `Memory-Allow-Origin` similar to [`Timing-Allow-Origin`](https://w3c.github.io/resource-timing/#sec-timing-allow-origin) or [`CORP: cross-origin`](https://github.com/whatwg/html/issues/4175#issuecomment-482734751).
- **[slow]** *per-frame sizes are requested in a window agent*: require either heap iteration, or heap segregation by origin, or accounting on allocation, or throwing a `NotSupportedError` exception.

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

## See also

The proposal was presented at [WebPerf WG F2F June 2019](https://docs.google.com/document/d/12ANc7fbKpjs__Qw_0DxM74u49276vTwRwCPyBxUkfBw/edit#heading=h.nraz045xllk0) meeting.
Notes, slides, video are available [here](https://docs.google.com/document/d/1uQ7pXwuBv-1jitYou7TALJxV0tllXLxTyEjA2n1mSzY/edit#bookmark=id.sagvdwcwhq1h).
