## Explainer: Cross-origin opener policy reporting API

### Overview

We want to provide a reporting API for cross-origin opener policy (COOP) to
help developers deploy it on their websites. In addition to reporting breakages
when COOP is enforced, we want to provide a report-only mode for COOP. The
report-only mode for COOP will not enforce COOP, but it will report potential
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

> This means that setting both HTTP headers "Cross-Origin-Opener-Policy-Report-Only=same-origin" and "Cross-Origin-Embedder-Policy-Report-Only=require-corp" will lead the document to have a report only COOP value of *same-origin-plus-coep*. We do this to help facilitate deployment of both COOP and COEP: if developers want to eventually have both, they do not need to choose which one should move out of report-only mode first.

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
	- Otherwise, it's the initial navigation URL (because the COOP page is the opener so it is the initiator of the navigation).
- **Other documents in the browsing context group URL for reporting** - this corresponds to the URL of the other documents in the browsing context group. To report it safely, we do the following:
	- If the other document, the current document and their respective redirect chains are all same-origin, this is the URL of the other document.
	- Otherwise, it's the empty URL.

### Reporting browsing context switches

The first type of violation we report are browsing context group switches. This
is only useful if the navigating browsing context has another top level
browsing context in its browsing context group. In this case, the browsing
context group switch caused by COOP would sever the communication between the
top-level browsing contexts. Otherwise, it has no impact. Thus we only report
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

If a redirect response specifies a *reporting endpoint* in its
Cross-Origin-Opener-Policy or Cross-Origin-Opener-Policy-Report-Only header and
its COOP *value* or *report only value* would cause a browsing context group
switch, we should also send a violation report to its endpoint (with the
redirect response's URL).

These reports can also be sent in report-only mode. In that case, we need to
compute that a browsing context group switch would have happened if we had
enforced the *report only value* of COOP. To do that when navigating to a page with report only COOP, we check if:
- the previous document COOP and the navigation report only COOP require a browsing context group switch
- the previous document report only COOP and the navigation report only COOP require a browsing context group switch

If both checks require a browsing context group switch, we send a violation
report for *browsing context group switch due to a navigation to the page with
COOP reporting*.

> This allows a website to set the same report only COOP on all its pages and not get a violation report when navigating from one to another.

Similarly, when navigating away from a page with report only COOP, we check if:
- the current document report only COOP and the navigation COOP require a browsing context group switch
- the current document report only COOP and the navigation report only COOP require a browsing context group switch

If both checks require a browsing context group switch, we send a violation
report for *browsing context group switch due to navigating away from the page
with COOP reporting*.

### Report blocked accesses to other windows by the COOP page

The next step in reporting is to add reporting of accesses to other windows
made by the COOP page that were blocked because COOP triggered a browsing
context group switch. Similarly, in report-only mode, we should report accesses
the page makes to other windows that would be blocked if COOP were enforced.

To identify when to send these reports, we need to add a set *browsingContextsToNotifyOfAccess* to top-level
**BrowsingContext**. It will contain pairs of *BrowsingContexts* and a boolean *reportOnly*.

Then we modify the **obtain a new browsing context** step defined in COOP (this
step happens when COOP triggers a Browsing Context Group switch):
1.  When the navigation's **cross-origin opener policy** has a *reporting endpoint* and *browsingContext* (the current browsing context) has an opener, we create *newBrowsingContext* with that opener and mark the opener as closed. We add *newBrowsingContext* to the opener's set of *browsingContextsToNotifyOfAccess*.
2. For all top-level **browsing contexts** in *browsingContext*'s *browsingContextGroup* we check if their **cross-origin opener policy** has a *reporting endpoint*. We add all those who do to *browsingContext*'s set of *browsingContextsToNotifyOfAccess*.
3. If *browsingContext*'s set of *browsingContextsToNotifyOfAccess* is not empty, instead of discarding it we only discard its document.

Step 1 above allows us to capture the COOP page's accesses to its opener. Step
2 and 3 allows to capture the COOP page's accesses to windows it opened but
that were placed in another *browsing context group*. Note that the marking
happens when the other window navigates.

In report-only mode, we only check if a browsing context group switch
would have happened if we enforced COOP. When this is the case:

1. If the navigation COOP has a *report only reporting endpoint*, add *browsing context* to its opener set of *browsingContextsToNotifyOfAccess*.
2. Add all top-level **browsing contexts** in the **browsing context group** that have a COOP *report-only reporting endpoint* to *browsingContext*'s set of *browsingContextsToNotifyOfAccess*.

After loading the page with COOP reporting, when the new browsing context
navigates to another page that is cross-origin or no longer has the same COOP (including reporting), we remove it from the set of
*browsingContextsToNotifyOfAccess* in all top-level browsing contexts. For each
browsing context, if its set of *browsingContextsToNotifyOfAccess* becomes
empty and it had been kept in step 3 above, we now discard it entirely.

>  Keeping the browsing context in the set of accesses to notify when
>  navigating to a same-origin page with the same COOP allows to report issues
>  when the first page of a site triggers the browsing context group switch,
>  but the blocked access happens only on the second page of the site.

From there, we modify **WindowProxy**'s property access. When trying to access a
property on **WindowProxy**, we will check if the environment's *top-level
browsing context* is inside the **WindowProxy**'s *top level browsing
context*'s set of *browsingContextsToNotifyOfAccess*. If it is, and the
environment is same origin with its *top-level browsing context*, then we
inform the environment *top-level document* that one access to the
**WindowProxy** was blocked.

> The same-origin check on the environment is there to not report accesses to other windows coming from cross-origin iframes.

The document should then report a **blocked access from the COOP page to another window**. To do that, we generate a report
for the COOP document URL, the current environment and the following body:

- *disposition*: either "enforce" or "reporting" (depending on whether we're in report-only mode)
- *effective policy*: the *value* or *report only value* of the COOP page
- *blocked window uri*: either the **opener document URL for reporting** or the **openee document URL for reporting**, as defined in the **Safe URLs for reporting** section
- *violation*: "access-from-coop-page"
- *source file*, *lineno*, *colno*: if the user agent is currently executing script and can extract a source file's URL, line number and column number from the global object, set those accordingly.

### Report blocked accesses from other windows to the COOP page

Finally, we would also like to report accesses to the COOP page that were
blocked/would have been blocked by the enforcement of COOP.
