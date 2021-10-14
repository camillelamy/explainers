# COOP same-origin-allow-popups + COEP = crossOriginIsolated

## A problem

Sites that wish to continue using SharedArrayBuffer must opt-into cross-origin isolation. Among other things, cross-origin isolation will prevent cross-origin popups from having access to their opener. This behavior ships today in Firefox, and Chrome aims to ship it as well in Chrome 92.

As part of crossOriginIsolation, websites must send a `Cross-Origin-Opener-Policy: same-origin` header. COOP `same-origin` prevents pages with different top-level origins from being able to communicate with each other. This breaks many OAuth or payment flows that rely on opening a cross-origin popup that will communicate back with the page through `window.postMessage` for example. APIs like [WebID](https://github.com/WICG/WebID/blob/main/README.md) or [WebPayments](https://github.com/w3c/webpayments/blob/gh-pages/proposals/arch2020.md) will eventually solve the issue by providing developers with a way to build robust OAuth or payment flows without pop-ups through browser mediation. However, these APIs are not there yet, and will require significant changes from OAuth/Payment flow providers and users. we would like to find a solution that helps websites deploy COOP without having to implement a lot of changes to their websites.

## A solution

To make crossOriginIsolation easier to deploy on sites with OAuth/payment flows relying on popups, we would like `Cross-Origin-Opener-Policy: same-origin-allow-popups` to also enable crossOriginIsolation when served with an appropriate `Cross-Origin-Embedder-Policy` header. This would introduce a new COOP mode, with a few restrictions compared to regular COOP same-origin-allow-popups. However, this mode would be crossOriginIsolated, while still having access to any popup it opens through `window.postMessage`.

### COOP same-origin-allow-popup-plus-coep

We propose introducing a new [CrossOriginOpenerPolicy value](https://html.spec.whatwg.org/multipage/origin.html#cross-origin-opener-policy-value): **same-origin-allow-popups-plus-coep**. This value requires the top-level document of a page to respond with a `Cross-Origin-Opener-Policy: same-origin-allow-popups` header and a `Cross-Origin-Embedder-Policy: require-corp` header (or the `Cross-Origin-Embedder-Policy: credentialless` header if available - see [Crendetiallessness](https://github.com/mikewest/credentiallessness) for more info).

This new COOP value has the same behavior as COOP `same-origin-allow-popups` when it comes to navigation. Navigating to a COOP `same-origin-allow-popups-plus-coep` page will cause a browsing context group switch unless the previous page is same-origin and has the same COOP. However, the page will be able to open and navigate popups to COOP `unsafe-none` pages, and retain communication with them.

### COOP same-origin-allow-popup-plus-coep and crossOriginIsolation

The above change means that we can now have Browsing Context Groups holding pages with different **cross-origin isolation modes**. We propose removing [cross-origin isolation mode](https://html.spec.whatwg.org/multipage/browsers.html#bcg-cross-origin-isolation) from Browsing Context groups, allowing both cross-origin isolated and non cross-origin isolated pages to share a Browsing Context Group. Any top-level level document that sets both `Cross-Origin-Opener-Policy` : `same-origin-plus-coep` or `same-origin-allow-popups-plus-coep` would live in an agent cluster that has a **cross-origin isolation mode** of `concrete` or `logical`.

> In practice, on a platform that supports crossOriginIsolation, this means that all browsing contexts in page with a COOP of `same-origin-allow-popups-plus-coep` have a *cross-origin isolation mode* of `concrete`. When the page opens a popup and navigates it towards a COOP `unsafe-none` document, all the browsing contexts in the popup will have a *cross-origin isolation mode* of `none`. Hence the browsing context group has a [cross-origin isolation mode](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-isolation-mode) of `mixed`.

Browsing Context Groups were a useful structure to store **cross-origin isolation mode** because it provided easy inheritance for new [auxiliary browsing contexts](https://html.spec.whatwg.org/multipage/browsers.html#auxiliary-browsing-context) and [child browsing contexts](https://html.spec.whatwg.org/multipage/browsers.html#child-browsing-context). We will now explicitely state the inheritances in the navigation flow, directly from agent clusters. Nested browsing contexts's agent clusters  will inherit their **cross-origin isolation mode** from their parent's document agent cluster, while auxiliary browsing contexts will inherit from their opener's document agent cluster.

### COOP same-origin-allow-popups-plus-coep and Agent Clusters

For COOP `same-origin-allow-popups-plus-coep` pages to be crossOriginIsolated, the browser must provide the right security guarantees that will enable access to the APIs gated behind crossOriginIsolation. There are two risks here:
1. [Spectre](https://www.w3.org/TR/post-spectre-webdev/). Spectre attacks are way more efficient in crossOriginIsolated contexts due to the presence of high resolution timers.
2. crossOriginIsolated APIs that do not respect the same-origin policy like [performance.measureUserAgentSpecificMemory](https://github.com/WICG/performance-measure-memory).

Fortunately, both issues can be solved by modifying the scoping of [Agent Clusters](https://tc39.es/ecma262/#sec-agent-clusters). We will make two changes:
1. Like in COOP `same-origin-plus-coep` pages, we plan on scoping Agent Clusters in COOP `same-origin-allow-popup-plus-coep` based on origin rather than site URL.
2. We will add a **cross-origin isolation mode** value to [Agent Cluster keys](https://html.spec.whatwg.org/multipage/webappapis.html#agent-cluster-key) for [origin-keyed agent clusters](https://html.spec.whatwg.org/multipage/origin.html#origin-keyed-agent-clusters), along the origin. When [requesting a similar-origin window agent](https://html.spec.whatwg.org/multipage/webappapis.html#obtain-similar-origin-window-agent), we will also take into account the *cross-origin isolation mode* of computed from the headers. This will be factored into the computation of Agent Cluster key, meaning that requesting an Agent Cluster for the same origin would still be given different Agent Clusters if they have different cross-origin isolation modes.

### Security considerations

Why does this solve our issues? First, crossOriginIsolated APIs that do not respect the same-origin policy are scoped to a page, per the [crossOriginIsolation threat model](https://arturjanc.com/coi-threat-model.pdf). This means that we can safely enable them on a COOP `same-origin-allow-popups-plus-coep` page, as the page cannot share any Agent Cluster with a non-cross origin isolated popup it opens.

Second, defending against Spectre requires putting documents in different processes. This is only possible if the documents cannot access each other synchronously and cannot share memory. In practice, this means that the document must belong to different Agent Clusters. By ensuring that Agent Clusters cannot be shared by documents on pages with different COOP status, we ensure that pages with different COOP statuses can be put in different processes, even without Out-of-Process-Iframes. This is an efficient mitigation against Spectre, meaning that we can safely enable high precision timers in COOP `same-origin-allow-popups-plus-coep` pages.

So let's take the example of a COOP `same-origin-allow-popups-plus-coep` page that opens a COOP `unsafe-none` popup. Both pages have different **cross-origin isolation mode**, but they are in the same browsing context group. Same-origin documents in two pages with different cross-origin isolation mode will not have synchronous access to each other, since they will not have the same Agent Cluster. However, because the browsing contexts of the documents are in the same browsing context group, they can use `windowProxy` cross-origin like `postMessage` to communicate with one another.

This has effects on cross-origin iframes embedded in a COOP `same-origin-allow-popups-plus-coep` page. Those iframes will not be able to use `document.domain`. Nor will they be able to interact synchronously with a same-origin popup they open. We believe this restrictions are ok, as cross-origin iframes embedded in a COOP `same-origin-allow-popups-plus-coep` page need to have set a COEP. This allows them to be loaded in a COOP `same-origin-plus-coep` which has stricter restrictions: `document.domain` is not usable either, and they cannot communicate at all with any popup they open (all of their popups are opened with rel no-opener).

Another important point is that APIs that relied on the same-origin policy, might now also need to consider crossing **cross-origin isolation modes**. An audit of all such places was carried out in [this spreadsheet](https://docs.google.com/spreadsheets/d/1e6LakHSKTD22XEYfULUJqUZEdLnzynMaZCefUe1zlRc/edit). What came out of it was that only `javascript:` navigations caused a threat, allowing a non cross-origin isolated page to execute arbitrary javascript in a same-origin non cross-origin isolated page and vice-versa. Named targetting and location interface access were also investigated, but we believe they add no extra security risk as long as `javascript:` navigations are not possible.

### Privacy considerations

The main privacy threat is the information available from a WindowProxy crossing the **cross-origin isolation mode** boundary:
1. A cross-origin isolated page accessing the proxy of a non cross-origin isolated page cannot really use any cross-origin isolation gated APIs to get anything extra than what was already possible.
2. A non cross-origin isolated page accessing the proxy of a cross-origin isolated page is something that couldn't be done previously. But we also believe it is the opening page responsibility to decide to expose this extra information to be able to use popup functionalities. It could also be possible to make certain WindowProxy functions not available accross **cross-origin isolation mode** boundaries but we do not think it would be worth the added complexity.



