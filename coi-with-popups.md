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
