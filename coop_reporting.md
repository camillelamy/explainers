## Explainer: Cross-origin opener policy reporting API

### Overview

We want to provide a reporting API for cross-origin opener policy (COOP) to
help developers deploy it on their websites. In addition to reporting breakages
when COOP is enforced, we want to provide a report-only mode for COOP. The
report-only mode for COOP will not enforce COOP but it will report potential
breakages that would have happened had we enforced COOP.

### Changes to the Cross-Origin-Opener-Policy header

The Cross-Origin-Opener-Policy header is now defined as follows:

> The Cross-Origin-Opener-Policy HTTP response header is a Structured Header whose value must be a token.
>
> Valid Cross-Origin-Opener-Policy values include "unsafe-none", "same-origin-allow-popups" and "same-origin". These values may have a parameter specifying a string which represents the endpoint for violation reporting.

We also define a new header, Cross-Origin-Opener-Policy-Report-Only:

> The Cross-Origin-Opener-Policy-Report-Only HTTP response header is a Structured Header whose value must be a token.
>
> Valid Cross-Origin-Opener-Policy-Report-Only values include "unsafe-none", "same-origin-allow-popups" and "same-origin". These values may have a parameter specifying a string which represents the endpoint for violation reporting.

Example of usage a reporting endpoint:

> Cross-Origin-Opener-Policy = same-origin-allow-popups; report-to="http://example.com" -> enforces a COOP of same-origin-allow-popups and sends violation reports to http://example.com.
>
> Cross-Origin-Opener-Policy-Report-Only = same-origin-allow-popups; report-to="http://example.com" -> does not enforce COOP, but send violation reports to http://example.com reporting breakages that would have happened if a COOP policy of same-origin-allow-popups had been enforced. 

### Changes to cross-origin opener policy

Cross-origin opener policy is in process of being merged into the HTML spec in
this
[PR](https://github.com/whatwg/html/pull/5334#pullrequestreview-410851665). We
modify the spec in the following way: cross-origin opener policy is no longer a
single value, but a struct composed of:

1. A **cross-origin opener policy value** *value*
2. A **reporting endpoint** *reporting endpoint*
3. A **cross-origin opener policy value** *report only value*
4. A **reporting endpoint** *report only reporting endpoint*

The **cross-origin opener policy value** has the following values:
1. *unsafe-none*
2. *same-origin-allow-popups*
3. *same-origin*
4. *same-origin-plus-coep*

We continue computing the **cross-origin opener policy value** *value* as currently defined. For the *report only value*, we do the following:

1. If the *Cross-Origin-Opener-Policy* header is absent or cannot be parsed, then return *unsafe-none*.
2. If the *Cross-Origin-Opener-Policy* header's bare item is "unsafe-none", then return *unsafe-none*.
3. If the *Cross-Origin-Opener-Policy* header's bare item is "same-origin-allow-popups", then return *same-origin-allow-popups*.
4. If the *Cross-Origin-Opener-Policy* header's bare item is "same-origin", then:
	1. If the response's **cross-origin embedder policy** *value* is *require-corp* or its *report only value* is *require-corp*, then return *same-origin-plus-coep*.
	2. Otherwise, return *same-origin*.

> This means that setting both HTTP headers "Cross-Origin-Opener-Policy-Report-Only=same-origin" and "Cross-Origin-Embedder-Policy-Report-Only=require-corp" will lead the document to have a report only COOP value of *same-origin-plus-coep*. We do this to help facilitate deployment of both COOP and COEP: if developers want to eventually have both, they do not need to chose which one should move out of report-only mode first.

### Safe URLs for reporting

Since COOP takes effect during navigation, violation reports will need to
include information about other documents to be useful for developers. This
could potentially leak private information, so we cannot simply expose the URL
of other documents. In addition to stripping URLs of hostnames and passwords as
normally done in reporting, we define the following as safe to include in a
report:

- **Previous document URL for reporting** - this corresponds to the URL of the document that navigated to the page with COOP reporting. To report it safely, we do the following:
	- If the current document and all its redirect chain are same-origin with the previous document, this is the previous document URL.
	- Otherwise, it's the referrer of the navigation.
- **Next document URL for reporting** - this corresponds to the URL of the document the page with COOP reporting is navigating to. To report it safely, we do the following:
	- If the next document and all its redirect chain are same-origin with the current document, this is the next document URL.
	- If the current document is the initiator of the navigation, then it's the initial navigation URL.
	- Otherwise, it's the empty URL.
- **Opener document URL for reporting** - this corresponds to the URL of the document that opened the page with COOP reporting. To report it safely, we do the following:
	- If the current document and all its redirect chain are same-origin with the opener document, this is the previous document URL.
	- Otherwise, it's the referrer of the navigation.
- **Openee document URL for reporting** - this corresponds to the URL of the documents opened by the page with COOP reporting. To report it safely, we do the following:
	- If the document opened by the current document and all its redirect chain are same-origin with the current document, this is the opened document URL.
	- Oherwise, it's the initial navigation URL (because the COOP page is the opener so it is the initator of the navigation).
- **Other documents in the browsing context group URL for reporting** - this corresponds to the URL of the other documents in the browsing context group. To report it safely, we do the following:
	- If the other document, the current document and their respective redirect chains are all same-origin, this is the URL of the other document.
	- Otherwise, it's the empty URL.

### Reporting browsing context switches

The first type of violation we report are browsing context group switches. This
is only useful if the navigating browsing context has another top level
browsing context in its browsing context group. In this case, the browsing
context group switch caused by COOP would sever the communication between the
top-level browsing contexts. Otheriwse, it has no impact. Thus we only report
browsing context group switches when there is more than 1 top level browsing
context in the browsing context group.

When reporting a **browsing context group switch due to a navigation to the
page with COOP reporting**, we generate a report for the COOP document URL and
the following body:

- *disposition*: either "enforce" or "reporting" (depending on whether we're in report-only mode)
- *effective policy*: the *value* or *report only value* of the COOP page
- *navigation uri*: the **previous document URL for reporting**, as defined in the **Safe URLs for reporting** section
- *violation*: "navigate-to-document"

When reporting a **browsing context group switch due to a navigation away from
page with COOP reporting**, we generate a report for the COOP document URL and
the following body:

- *disposition*: either "enforce" or "reporting" (depending on whether we're in report-only mode)
- *effective policy*: the *value* or *report only value* of the COOP page
- *navigation uri*: the **next document URL for reporting**, as defined in the **Safe URLs for reporting** section
- *violation*: "navigate-from-document"

> Note that *effective policy* can be *same-origin-plus-coep* even though this value cannot be set through the Cross-Origin-Opener-Policy header alone.

This reports can also be sent in report-only mode. 
