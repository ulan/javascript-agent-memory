# Self-Review Questionnaire: Security and Privacy of JavaScript Memory API

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

The feature provides a `performance.measureMemory()` function that computes the size of JavaScript
objects of the same-origin realms of the current JavaScript agent.
See [the explainer](https://github.com/ulan/javascript-agent-memory/blob/master/explainer.md) for explanation why we need this feature.

Besides that there is no exposure of information.

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?

Yes.

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

No personal information is involved.

### 2.4. How does this specification deal with sensitive information?

No sensitive information is involved.

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?

No.

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

The origin can infer whether the underlying platform is 32-bit or 64-bit:
the result of the API on 64-bit systems will be larger than that on 32-bit systems.

The origin can alrady [get](https://stackoverflow.com/questions/1741933/detect-64-bit-or-32-bit-windows-from-user-agent-or-javascript) that information now from `window.navigator`. 

### 2.7. Does this specification allow an origin access to sensors on a user’s device

No.

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

The feature provides a `performance.measureMemory()` function that computes the size of JavaScript
objects of the same-origin realms of the current JavaScript agent.

Size information does not leak to different origins.


### 2.9. Does this specification enable new script execution/loading mechanisms?

No.

### 2.10. Does this specification allow an origin to access other devices?

No.

### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?

No.

### 2.12. What temporary identifiers might this this specification create or expose to the web?

None.

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

No.

### 2.14. How does this specification work in the context of a user agent’s Private \ Browsing or "incognito" mode?

It works the same in all modes.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Yes, see [the explainer](https://github.com/ulan/javascript-agent-memory/blob/master/explainer.md).
Note that security and privacy considerations are the same for this API.

### 2.16. Does this specification allow downgrading default security characteristics?

No.

### 2.17. What should this questionnaire have asked?

No idea of other questions.