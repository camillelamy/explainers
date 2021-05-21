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

We also propose a new [cross-origin isolation mode](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-isolation-mode): `mixed`. When loading a COOP same-origin-allow-popups-plus-coep page in a browsing context group, we will switch its cross-origin isolation mode to `mixed`. This reflects that some of the browsing contexts in the browsingContextGroup might be crossOriginIsolated, while others won't be.

To support this, we plan on adding the notion of **cross-origin isolation mode** to browsing contexts as well. This will be set on top-level browsing contexts based on the COOP of the page they navigate to:
* `logical` if the page has a COOP of `same-origin-plus-coep` or `same-origin-allow-popups-plus-coep`, and the platform cannot support the crossOriginIsolated security guarantees.
* `concrete` if the page has a COOP of `same-origin-plus-coep` or `same-origin-allow-popups-plus-coep`, and the platform can support the crossOriginIsolated security guarantees.
* `none` otherwise.

Child browsing contexts will inherit their **cross-origin isolation mode** from their parent.

> In practice, on a platform that supports crossOriginIsolation, this means that all browsing contexts in page with a COOP of `same-origin-allow-popups-plus-coep` have a *cross-origin isolation mode* of `concrete`. When the page opens a popup and navigates it towards a COOP `unsafe-none` document, all the browsing contexts in the popup will have a *cross-origin isolation mode* of `none`. Hence the browsing context group has a [cross-origin isolation mode](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-isolation-mode) of `mixed`.

### COOP same-origin-allow-popups-plus-coep and Agent Clusters

For COOP `same-origin-allow-popups-plus-coep` pages to be crossOriginIsolated, the browser must provide the right security guarantees that will enable access to the APIs gated behind crossOriginIsolation. There are two risks here:
1. [Spectre](https://www.w3.org/TR/post-spectre-webdev/). Spectre attacks are way more efficient in crossOriginIsolated contexts due to the presence of high resolution timers.
1. crossOriginIsolated APIs that do not respect the same-origin policy like [performance.measureUserAgentSpecificMemory](https://github.com/WICG/performance-measure-memory).

Fortunately, both issues can be solved by modifying the scoping of [Agent Clusters](https://tc39.es/ecma262/#sec-agent-clusters). We will make two changes:
1. Like in COOP `same-origin-plus-coep` pages, we plan on scoping Agent Clusters in COOP `same-origin-allow-popup-plus-coep` based on origin rather than site URL.
2. We will add a **cross-origin isolation mode** value to [Agent Cluster keys](https://html.spec.whatwg.org/multipage/webappapis.html#agent-cluster-key), in addition to the Site URL or the origin. When [requesting a similar-origin window agent](https://html.spec.whatwg.org/multipage/webappapis.html#obtain-similar-origin-window-agent), we will also take into account the *cross-origin isolation mode* of the browsing context requesting the Agent Cluster, not just of the browsing context group. This will be factored into teh computation of Agent Cluster key, meaning that browsing contexts requesting an Agent Cluster for the same origin would still be given different Agent Clusters if they have different cross-origin isolation modes.

Why does this solve our issues? First, crossOriginIsolated APIs that do not respect the same-origin policy are scoped to an Agent Cluster, per the [crossOriginIsolation threat model](https://arturjanc.com/coi-threat-model.pdf). This means that we can safely enable them on a COOP `cross-origin-allow-popups-plus-coep` page, as the page cannot share any Agent Cluster with a non-cross origin isolated popup it opens.

Second, defending against Spectre requires putting documents in different processes. This is only possible if the documents cannot access each other synchronously and cannot share memory. In practice, this means that the document must belong to different Agent Clusters. By ensuring that Agent Clusters cannot be shared by documents on pages with different COOP status, we ensure that pages with different COOP statuses can be put in different processes, even without Out-of-Process-Iframes. This is an efficient mitigation against Spectre, meaning that we can safely enable high precision timers in COOP `same-origin-allow-popups-plus-coep` pages.

So let's take the example of a COOP `same-origin-allow-popups-plus-coep` page that opens a COOP `unsafe-none` popup. Both pages have different cross-origin isolation mode, but they are in the same browsing context group. Same-origin documents in two pages with different cross-origin isolation mode will not have synchronous access to each other, since they will not have the same Agent Cluster. However, because the browsing contexts of the documents are in the same browsing context group, they can use `windowProxy` cross-origin like `postMessage` to communicate with one another.

This has effects on cross-origin iframes embedded in a COOP `same-origin-allow-popups-plus-coep` page. Those iframes will not be able to use `document.domain`. Nor will they be able to interact synchronously with a same-origin popup they open. We believe this restrictions are ok, as cross-origin iframes embedded in a COOP `same-origin-allow-popups-plus-coep` page need ot have set a COEP. This allows them to be loaded in a COOP `same-origin-plus-coep` which has stricter restrictions: `document.domain` is not usable either, and they cannot communicate at all with any popup they open (all of their popups are opened with rel no-opener).
