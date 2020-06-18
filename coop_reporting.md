# Explainer: Cross-origin opener policy reporting API

## Overview

We want to provide a reporting API for cross-origin opener policy (COOP) to
help developers deploy it on their websites. In addition to reporting breakages
when COOP is enforced, we want focus on providing a report-only mode for COOP. The
report-only mode for COOP will not enforce COOP, but it will report potential
breakages that would have happened had we enforced COOP.

## Changes to the Cross-Origin-Opener-Policy header

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

## Changes to cross-origin opener policy

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

1. If the *Cross-Origin-Opener-Policy-Report-Only* header is absent or cannot be parsed, then return *unsafe-none*.
2. If the *Cross-Origin-Opener-Policy-Report-Only* header's bare item is "unsafe-none", then return *unsafe-none*.
3. If the *Cross-Origin-Opener-Policy-Report-Only* header's bare item is "same-origin-allow-popups", then return *same-origin-allow-popups*.
4. If the *Cross-Origin-Opener-Policy-Report-Only* header's bare item is "same-origin", then:
	1. If the response's **cross-origin embedder policy** *value* is *require-corp* or its *report only value* is *require-corp*, then return *same-origin-plus-coep*.
	2. Otherwise, return *same-origin*.

> This means that setting both HTTP headers "Cross-Origin-Opener-Policy-Report-Only=same-origin" and "Cross-Origin-Embedder-Policy-Report-Only=require-corp" will lead the document to have a report only COOP value of *same-origin-plus-COEP*. We do this to help facilitate deployment of both COOP and COEP: if developers want to eventually have both, they do not need to choose which one should move out of report-only mode first.

## Safe URLs for reporting

Since COOP takes effect during navigation, violation reports will need to
include information about other documents to be useful for developers. This
could potentially leak private information, so we cannot simply expose the URL
of other documents. In addition to stripping URLs of usernames and passwords as
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

## Reporting browsing context switches

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
- *effective-policy*: the *value* or *report only value* of the COOP page
- *navigation-url*: the **previous document URL for reporting**, as defined in the **Safe URLs for reporting** section
- *violation*: "navigate-to-document"

When reporting a **browsing context group switch due to a navigation away from
page with COOP reporting**, we generate a report for the COOP document URL and
the following body:

- *disposition*: either "enforce" or "reporting" (depending on whether we're in report-only mode)
- *effective-policy*: the *value* or *report only value* of the COOP page
- *navigation-url*: the **next document URL for reporting**, as defined in the **Safe URLs for reporting** section
- *violation*: "navigate-from-document"

> Note that *effective-policy* can be *same-origin-plus-COEP* even though this value cannot be set through the Cross-Origin-Opener-Policy header alone.

### Popups

When a document is same-origin with the top-level document, any popup they open inherits the cross-origin opener policy of the top-level document. In this case, when the browsing context group switch happens durign the first navigation of the popup, we should use the opener of the popup in lieu of the previous document in the preceding algorithms.

> Otherwise, we'd generate reports for the intial about:blank document, which would not be very useful.

This does not apply to popups created by a cross-origin iframe, which do not inherit COOP from their opener.
In particular, we cannot report that a popup was opened with rel-noopener due to COOP.
This would give too much information about the behavior of cross-origin frames.
Depending on who would enable reporting, we would leak:

- that a cross-origin iframe tried to open a popup to the parent document having enabled COOP reporting
- that the iframe was embedded in a "*same-origin*" COOP document to the iframe having enabled COOP reporting

### Redirects

If a redirect response specifies a *reporting endpoint* in its
Cross-Origin-Opener-Policy or Cross-Origin-Opener-Policy-Report-Only header and
its COOP *value* or *report only value* would cause a browsing context group
switch, we should also send a violation report to its endpoint (with the
redirect response's URL).

### Report-only mode

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

## Report blocked accesses to other windows

The next step in reporting is to add reporting of accesses to other windows
that were/would be blocked because COOP triggered a browsing
context group switch. Unfortunately, this is only doable easily in report-only mode, so we focus on providing that.

### Browsing context changes

#### Monitoring accesses

To identify when to send these reports, we need extra bookeeping on **BrowsingContexts**.

We define the following **COOPAccessMonitor** struct:

- A **BrowsingContext** *browsingContext* whose document has enabled COOP reporting
- A string *report-type* whose value is either "*report-accesses-from*" or "*report-accesses-to*"

Then we add a set of these **COOPAccessMonitors**, *browsingContextsToNotifyOfAccess*, to top-level
**BrowsingContexts**.

In report-only mode, we only check if a browsing context group switch
would have happened if we enforced COOP. When this is the case:

1. We need to check if we need to monitor accesses between this window and other windows that would have been in another browsing context group had coop been enforced. This is the case if *browsingContext* (the current browsing context) has an opener and the navigation's **cross-origin opener policy** has a *report-only reporting endpoint* or or *browsingContext*'s opener is same-origin with its top-level browsing context and the opener's top-level browsing context's **cross-origin opener policy** has a *report-only reporting endpoint*. If there is no need to monitor access, we can proceed normally.
2. If the navigation's **cross-origin opener policy** has a *report only reporting endpoint*
	1. We add *browsingContext* to its opener set of *browsingContextsToNotifyOfAccess* (with *report-accesses-from*).
	2. We add *browsingContext* to *browsingContext*'set of *browsingContextsToNotifyOfAccess* (with *report-accesses-to*).
3.  If *browsingContext*'s opener is same-origin with its top level browsing context and the opener's top-level browsing context's **cross-origin opener policy** has a *report only endpoint*:
	1. We add *browsingContext*'s opener's top-level browsing context to *browsingContext*'s set of *browsingContextsToNotifyOfAccess* (with *report-accesses-from*).
	2. We add *browsingContext*'s opener's top-level browsing context to *browsingContext*'s opener's set of *browsingContextsToNotifyOfAccess* (with *report-accesses-to*).

Step 2.1 allows us to capture accesses made by the COOP report-only page to its opener.
Step 2.2 allows us to capture accesses made to the COOP report-only page by other windows in the browsing context group.

![Reporting when navigating to a COOP report-only page](/COOP_reporting_only_being_opened.png)

Step 3.1 allows us to capture accesses made by a COOP report-only pages to other window it opens.
Step 3.2 allows us to capture accesses made to a COOP report-only pages by other window it opens.

![Reporting when opening a window from a COOP report-only page](/COOP_reporting_only_opening.png)

After loading the page with COOP reporting, when the new browsing context
navigates to another page that is cross-origin or no longer has the same COOP (including reporting),
we remove it from the set of
*browsingContextsToNotifyOfAccess* in all top-level browsing contexts.

>  Keeping the browsing context in the set of accesses to notify when
>  navigating to a same-origin page with the same COOP allows to report issues
>  when the first page of a site would trigger the browsing context group switch,
>  but the blocked access would only happen on the second page of the site.

#### Virtual browsing context group id

In report-only mode, monitoring accesses is not enough to distinguish accesses from other windows that would be blocked (because they would be in another browsing context group) from those that wouldn't be blocked (because they would be in the same browsing context group). To correctly assess those, we introduce the notion of *virtualBrowsingContextGroupId* to browsing contexts. Whenever we detect that the enforcement of a report-only COOP policy would have resulted in a browsing context group switch, we assign a new *virtualBrowsingContextGroupId* to the browsing context that is navigating. Whenever a new top-level browsing context is created, it inherits its *virtualBrowsingContextGroupId* from its creator. This allows us to easily know which browsing contexts would be in which different browsing context groups if COOP were enforced.

*The* virtualBrowsingContextGroupId *should change when encountering pages with COOP report-only when the COOP would normally lead to a browsing context group switch, as shown in the diagrams below.*

![Virtual browsing context group id 1](/COOP_virtual_bcg1.png)

![Virtual browsing context group id 2](/COOP_virtual_bcg2.png)

![Virtual browsing context group id 3](/COOP_virtual_bcg3.png)

![Virtual browsing context group id 4](/COOP_virtual_bcg4.png)

### WindowProxy changes

From there, we modify **WindowProxy**'s property access. When trying to access a
property on **WindowProxy** as part of the **[[Get]]** or **[[Set]]** operations, we will check the **COOPAccessMonitors** of **WindowProxy**'s *top level browsing context*'s *browsingContextsToNotifyOfAccess*:
1. For all **COOPAccessMonitors** with a *report-type* of *report-access-to*:
	1. If **WindowProxy**'s *top level browsing context*'s *virtualBrowsingContextGroupId* is the same as the **incumbent global object**'s *top-level browsing context*'s *virtualBrowsingContextGroupId*, proceed.
	2. If the property is not part of the **cross-origin properties**, proceed.
	3. Otherwise inform the *browsingContext* in the **COOPAccessMonitors** of a **blocked access to the COOP page from another window**, given the **incumbent global object**'s *top-level browsing context*, and the property being accessed.
2. If there is a **COOPAccessMonitor** whose *browsingContext* is the environment's *top-level browsing context* and its *report-type* is *report-access-from*:
	1. If **WindowProxy**'s *top level browsing context*'s *virtualBrowsingContextGroupId* is the same as the **incumbent global object**'s *top-level browsing context*'s *virtualBrowsingContextGroupId*, proceed.
	2. If the **incumbent global object** is not same origin with its *top-level browsing context*, proceed.
	3. If the property is not part of the **cross-origin properties**, proceed.
	4. Otherwise, inform the **incumbent global object**'s *top-level document* of a **blocked access from the COOP page to another window**, given the **WindowProxy**'s *top level browsing context*, the environment and the property being accessed.

> The same-origin check on the environment is there to not report accesses to other windows coming from cross-origin iframes.

> We only send violation reports for accesses to cross-origin properties, as we think websites will deploy COOP on all pages coming from the same origin, meaning that cross-origin accesses to cross-origin properties is where the bulk of the violations will occur.

### Emit reports

When the document is notifed of a **blocked access from the COOP page to another window**, it should generate a report
for the COOP document URL, the current environment and the following body:

- *disposition*: "reporting"
- *effective-policy*: the *value* or *report only value* of the COOP page
- *blocked-window-url*:
	- the **opener document URL for reporting** if the COOP page is trying to access its opener
	- the **openee document URL for reporting** if the COOP page is trying to access a page it opened
	- the **other documents in the browsing context group URL for reporting** in other cases
	> All are defined in the **Safe URLs for reporting** section.
- *violation*: "access-from-coop-page"
- *property*: the name of the property being accessed
- *source-file*, *lineno*, *colno*: if the user agent is currently executing script and can extract a source file's URL, line number and column number from the global object, set those accordingly.

> The report for blocked access from the COOP page should also notify ReportingObservers, unlike the rest of the reports described in this page.

When the document is notifed of a **blocked access to the COOP page from another window**, it should generate a report for the COOP document URL, the current environment and the following body:

- *disposition*: "reporting"
- *effective-policy*: the *value* or *report only value* of the COOP page
- *blocked-window-url*:
	- the **opener document URL for reporting** if the window trying to access the COOP page is its opener
	- the **openee document URL for reporting** if the window trying to access the COOP page was opened by the COOP page
	- the **other documents in the browsing context group URL for reporting** in other cases
	> All are defined in the **Safe URLs for reporting** section.
- *violation*: "access-to-coop-page-same-origin"
- *property*: the name of the property being accessed.

### Reporting blocked accesses when COOP is enforced (outside the scope of this proposal)

In order to report blocked accesses when COOP is enforced, we would need to do modify the **obtain a new browsing context** step defined in COOP (this step happens when COOP triggers a Browsing Context Group switch):

1. First, we check if there is a need to monitor accesses between this window and other windows in its fromer browsing context group. This is the case if *browsingContext* (the current browsing context) has an opener and the navigation's **cross-origin opener policy** has a *reporting endpoint* or *browsingContext*'s opener is same-origin with its top-level browsing context and the opener's top-level browsing context's **cross-origin opener policy** has a *reporting endpoint*. If there is no need to monitor access, we can proceed normally.
2. We create *newBrowsingContext* with an empty browsing context *newOpener* as opener and mark the opener as closed.
3. Instead of discarding *browsingContext*, we only discard its document.
4. If the navigation's **cross-origin opener policy** has a *reporting endpoint*:
	1. We add *newBrowsingContext* to the *newOpener*'s set of *browsingContextsToNotifyOfAccess* (with *report-only* false and *report-accesses-from*).
	2. We add *newBrowsingContext* to *browsingContext*'s set of *browsingContextsToNotifyOfAccess* (with *report-only* false and *report-accesses-to*).
5. If *browsingContext*'s opener is same-origin with its top level browsing context and the opener's top-level browsing context's **cross-origin opener policy** has a *reporting endpoint*:
	1. We add *browsingContext*'s opener's top level browsing context to *browsingContext*'s set of *browsingContextsToNotifyOfAccess* (with *report-only* false and *report-accesses-from*).
	2. We add *browsingContext*'s opener's top level browsing context to *newOpener*'s set of *browsingContextsToNotifyOfAccess* (with *report-only* false and *report-accesses-to*).

Step 4.1 above allows us to capture the COOP page's accesses to its opener.
Step 4.2 allows us to capture the accesses made to a COOP page from windows in its former browsing context group.

![Reporting when navigating to a COOP page](/COOP_reporting_being_opened.png)

Step 5.1 allows us to capture the COOP page's accesses to windows it opened but
that were placed in another *browsing context group*. Note that the marking
happens when the other window navigates.
Step 5.2 allows us to capture the accesses made to a COOP page from windows it opened but that were placed in its former browsing context group. Again, the marking happens when the other window navigates.

![Reporting when opening a window from a COOP page](/COOP_reporting_opening_window.png)

From there, we can reuse the changes described in the **WindowProxy changes** section above to properly emit reports.

However, all of this requires creating a dummy opener object. We believe this too complex for the benefit of getting reports in enforcement mode, so we are not currently looking to implement this.

## Security and privacy considerations

One of the principal risks of introducing the COOP reporting API is that the reports could leak information about cross-origin frame or window behaviors to the document that enables COOP reporting. To avoid this, we have included specific mitigations:
1. The URLs of other documents that we report are sanitized so that they provide no more information about navigation than what the page would normally know.
2. Cross-origin iframe's action are not included in the COOP reports sent to the endpoint specified by the top-level page.

That said, the proposal gives more information than currently available in the reports for blocked accesses to a page with COOP reporting. In particular, it informs the COOP page that a cross-origin window tried to access a particular property on its window, and it add information about the cross-origin URL (referrer URL or initial navigation URL). We think this information is reasonnable to surface, as the access would normally result in JavaScript execution, which could potentially be detected by the page anyway.

Finally, all information sent to the reporting endpoint goes through the reporting API, which provides security and privacy guarantees regarding the data collected in the reports.

Answers for the TAG review security and privacy questionaire:

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

When a page enables this feature, it can know that other cross-origin pages tried to access cross-origin properties of its window. This allows it to know that enforcing COOP would break such behaviors.

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?

Yes.

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

This API does not currently expose any PII information.

### 2.4. How does this specification deal with sensitive information?

This API does not currently expose sensitive information.

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?

No.

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

None.

### 2.7. Does this specification allow an origin access to sensors on a user’s device?

No.

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

An origin can know that a window it opened has COOP or COOP report-only enabled by getting report that accesses to it are blocked. The fact that a window had COOP enforced could already be inferred.

The feature also exposes the fact that other origins tried to access particular cross-origin properties of the window of the document that enabled COOP reporting. The document could already know that the properties were accessed, however this feature correlates that with a URL. This URL is already known to the page, and is not necessarily the current URL of document trying to access the cross-origin property.

### 2.9. Does this specification enable new script execution/loading mechanisms?

No.

### 2.10. Does this specification allow an origin to access other devices?

No.

### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?

No.

### 2.12. What temporary identifiers might this this specification create or expose to the web?

No.

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

This feature can only be enabled by a top-level navigation. When enabled, it applies to all actions in same-origin documents or targetting same-origin documents. Cross-origin iframes are not covered by the reporting API. All information is sent to a reporting endpoint specified by the top-level document, which cannot be changed by a third-party.

### 2.14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?

The behavior should be the same as for regular mode.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Yes, located above the answers to this present questionaire.

### 2.16. Does this specification allow downgrading default security characteristics?

No.

## Limitations of the API

The API does not report blocked accesses to/from the COOP window when COOP is enforced. This kind of report is only available in report-only mode. In particular, the reports are only emitted for breakages that would happen if the report-only value of COOP was enforced compared to the regular value of COOP.

> So if a website sets COOP same-origin-allow-popups and report-only COOP same-origin, they wouldn't receive reports for breakages due to the enforcement for the same-origin-allow-popups policy. However, they would get reports for the additional breakages that would occur if we were to enforce a policy of same-origin.

The API as defined currently does not give a way for cross-origin subframes embedded in a COOP page to detect that they have been broken by their parent COOP. There are privacy and security considerations to emitting reports bound for subframes based on a policy their cross-origin parent set. At the same time, there is no good way for an iframe to signal that it does not want to be embedded in a particular COOP environment, so it's not clear how actionable the reports would be anyway. Should we offer such a mechanism, we should extend the COOP reporting API to provide meaningful information to cross-origin iframes.
