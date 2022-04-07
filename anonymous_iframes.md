# Anonymous iframes

- **Author**: clamy@google.com, arthursonzogni@google.com
- **Created**: 2021-05-06
- **Last Updated**: 2022-04-07

The content has been moved into its own repository:
https://github.com/ArthurSonzogni/anonymous-iframe

<details>

   <summary>Access old content</summary>
   

## Table of content
- [A problem](#a-problem)
- [A proposal](#a-proposal)
- [Security and privacy considerations](#security-and-privacy-considerations)
- [Alternatives considered](#alternatives-considered)
- [Self-Review Questionnaire: Security and Privacy](#self-review-questionnaire--security-and-privacy)

## A problem

Deploying COEP is difficult for some developers, because of 3rd party iframes.
This is the typical scenario:

1. End users needs performant websites.
2. Some developers get performant website, by using
   multithreading/SharedArrayBuffer on the main document.
3. To mitigate Spectre attacks: Chrome, Firefox and Safari
   gate SharedArrayBuffer usage behind the [crossOriginIsolated][coi]
   capability. This requires [COOP and COEP][coop-coep].
4. COEP requirement is recursive: 3rd party iframes are required to deploy COEP
   in order to be embeddable inside a COEP parent.
5. Waiting for 3rd party to deploy COEP is painful for developers. This is out
   of their control most of the time.

Outside of performance, there are additionnal reasons website want to deploy
COEP: Getting the [cross-origin-isolated capability][coi], [high resolution
timers][highres-timer], [getViewportMedia][getViewPortMedia], etc...

[coi]: https://html.spec.whatwg.org/#concept-settings-object-cross-origin-isolated-capability
[highres-timer]: https://www.w3.org/TR/hr-time/#:~:text=if%20crossoriginisolatedcapability%20is%20true
[getViewPortMedia]: https://github.com/w3c/mediacapture-screen-share/issues/155#issuecomment-812269225
[coop-coep]: https://web.dev/coop-coep/
[coep-credentialless]:https://wicg.github.io/credentiallessness/

Deploying COEP is challenging in cases where there's not a single developer
involved, but many. Google Ads, for example, includes third-party content, and
it seems somewhat unlikely that they'll be able to ensure that all the ads
creators will do the work to opt-into being loadable.

## A proposal

It would be ideal if we could find an approach that provided robust-enough
protection against accidental cross-process leakage without requiring an
explicit opt-in.

COEP, and the opt-in mechanism for [getViewportMedia][getViewportMedia]
currently tackles data leak attacks by ensuring that cross-origin resources
explicitly opt into being loaded in an environment with higher risks. This way,
servers can protect vulnerable resources by not having them opt into being
loaded in high risk environments. [COEP:credentialless][coep-credentialless] and
the mechanism we are proposing, anonymous iframes, take a different approach.
Instead of requiring opt-in to protect vulnerable resources, we aim to only make
requests that result in public data being loaded in high-risk environments.
Essentially, this means preventing personalized resources from being loaded.

### What are anonymous iframes?

Documents can create anonymous iframes by adding an attribute to the iframe tag:

`<iframe anonymous src=”example.com”>`

This property is stored on the Browsing Context. It is inherited by any child
browsing context of the anonymous iframe, e.g.:

![](https://docs.google.com/drawings/d/e/2PACX-1vQDqWv-shIWWZ3gvnM51mQrN-DkcXPDQEZKkpEtT7PKMJa8fBUVFE__RZkI-LKbrhqj8pnnmVmTfAcD/pub?w=504&h=638)

Similar to sandbox flags, the attribute can be changed programmatically, but the change will only take effect on the next navigation taking place inside the iframe.

In parallel with the iframe attribute, we plan to add a new Fetch Metadata header to outgoing requests to inform servers of the COEP/anonymous context in which the response will be embedded:

* `Sec-Fetch-COEP: none`: the request will not be rendered in a COEP context, either because this is a top-level navigation or the frame in which the resource will be rendered in a context with a COEP of unsafe-none.
* `Sec-Fetch-COEP: require-corp`: the resource will be rendered in a context with a COEP of require-corp.
* `Sec-Fetch-COEP: credentialless`: the resource will be rendered in a context with a COEP of credentialless.
* `Sec-Fetch-COEP: anonymous`: the resource will be rendered in an anonymous iframe.

Additionally, we plan on adding a `window.anonymous` read-only attribute. By
default, this will return `false`. In anonymous iframes it will return `true`,
allowing a document to check whether it was loaded in an anonymous iframe.

### Anonymous iframes and credentials

Anonymous iframes cannot use existing credentials and shared storage for their origin. They are given a blank slate. Unlike sandboxed frames, they can use storage APIs and register cookies. However, those credentials and storage can only be shared by documents in anonymous iframes in the page (provided they meet origin restrictions). They will no longer be accessible once the page has navigated. Essentially, anonymous iframes are given a temporary [storage shelf](https://storage.spec.whatwg.org/#storage-shelf) partitioned to anonymous iframes in the page.

To achieve this, we rely on modifying the [storage key](https://storage.spec.whatwg.org/#storage-key) used to access shared storage by anonymous iframes. As part of the [client-side storage partitioning effort](https://privacycg.github.io/storage-partitioning/) (declined across [storage APIs](https://github.com/wanderview/quota-storage-partitioning/blob/main/explainer.md), [network state](https://github.com/MattMenke2/Explainer---Partition-Network-State) and [Cookie State](https://github.com/DCtheTall/CHIPS)), the storage key of an environment will no longer be its simple origin as currently described in the spec. Instead it will be a combination of the origin and the top-level site URL. In an anonymous iframe, we will replace the top-level site URL in the partition key by a nonce value, determined once per page. This nonce will be recomputed every time the top-level frame navigates. This ensures that anonymous iframes cannot share storage keys with non-anonymous iframes. Because the nonce is changed on every page navigation, anonymous iframes do not share storage across different pages, or across navigations.

![](https://docs.google.com/drawings/d/e/2PACX-1vQexnC7CIXMI9hmhaoJBIww7IZQ8W40Nx0vD_jqk5Go6YfIpa1ZjS7fVNT0fjc8RMScwg3jA9FDI-HL/pub?w=550&h=866)

*Storage and credentials are only shared among anonymous iframes, following normal site/origin access checks.*

![](https://docs.google.com/drawings/d/e/2PACX-1vQ2n_iCkOn9j5l7JAEu-h2trGtdXFma0sjreBiv_AYhdEECopoXPFPwUoIwROwIq9iSkdpNrI9dpRGT/pub?w=1196&h=431)

*Storage and credentials created by anonymous iframes are no longer accessible after the top level frame navigated away because the Storage key for anonymous iframes will have changed. This applies to top-level history navigations as well, meaning that when the page navigates away, anything stored by anonymous iframes can be cleared by the browser, unless the page was stored in a back-forward cache.*

Popups opened by anonymous iframes are not anonymous. However, we impose that popups opened by anonymous iframes are opened with rel = noopener set. This is done to prevent OAuth popup flows from being used in anonymous iframes. See the threat model part for a discussion on why we impose this restriction.

### How do anonymous iframes interact with COEP?

Our proposition is that anonymous iframes are safe enough to embed in a COEP page, even if they haven’t opted to do so by sending a COEP header. Thus, when navigating to a document in an anonymous iframe, we do not check whether it has a COEP and CORP header, even if its parent does not have a COEP of unsafe-none.

The [algorithm for enforcing COEP on a navigation response](https://html.spec.whatwg.org/multipage/origin.html#check-a-navigation-response's-adherence-to-its-embedder-policy) then becomes:

> To check a navigation response's adherence to its embedder policy given a [response](https://fetch.spec.whatwg.org/#concept-response) response, a [browsing context](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context) target, and an [environment](https://html.spec.whatwg.org/multipage/webappapis.html#environment) environment:
> 
> 1. If target is not a [child browsing context](https://html.spec.whatwg.org/multipage/browsers.html#child-browsing-context), then return true.
> 1. **If target is anonymous, then return true.**
> 1. Let responsePolicy be the result of [obtaining an embedder policy](https://html.spec.whatwg.org/multipage/origin.html#obtain-an-embedder-policy) from response and environment.
> 1. Let parentPolicy be target's [container document](https://html.spec.whatwg.org/multipage/browsers.html#bc-container-document)'s [embedder policy](https://html.spec.whatwg.org/multipage/dom.html#concept-document-embedder-policy).
> 1. If parentPolicy's [report-only value](https://html.spec.whatwg.org/multipage/origin.html#embedder-policy-report-only-value) is "`require-corp`" and responsePolicy's [value](https://html.spec.whatwg.org/multipage/origin.html#embedder-policy-value-2) is "`unsafe-none`", then [queue a cross-origin embedder policy inheritance violation](https://html.spec.whatwg.org/multipage/origin.html#queue-a-cross-origin-embedder-policy-inheritance-violation) with response, "`navigation`", parentPolicy's [report only reporting endpoint](https://html.spec.whatwg.org/multipage/origin.html#embedder-policy-report-only-reporting-endpoint), "`reporting`", and target's [container document](https://html.spec.whatwg.org/multipage/browsers.html#bc-container-document)'s [relevant settings object](https://html.spec.whatwg.org/multipage/webappapis.html#relevant-settings-object).
> 1. If parentPolicy's value is "unsafe-none" or responsePolicy's value is "require-corp", then return true.
> 1. [Queue a cross-origin embedder policy inheritance violation](https://html.spec.whatwg.org/multipage/origin.html#queue-a-cross-origin-embedder-policy-inheritance-violation) with response, "`navigation`", parentPolicy's [reporting endpoint](https://html.spec.whatwg.org/multipage/origin.html#embedder-policy-reporting-endpoint), "`enforce`", and target's [container document](https://html.spec.whatwg.org/multipage/browsers.html#bc-container-document)'s [relevant settings object](https://html.spec.whatwg.org/multipage/webappapis.html#relevant-settings-object).
> 1. Return false.

This also means that anonymous iframes can be embedded in cross-origin isolated pages without documents in them having to deploy COEP.

### Anonymous iframes and autofill/password managers

Browsers that implement autofill or password manager functionalities should make them unavailable in anonymous iframes. The goal of anonymous iframes is to preserve storage critical to an iframe function, but to avoid users logging into anonymous iframes. Autofill and password managers make logging in easier, and so should be avoided to prevent users accidentally logging in. This also allows anonymous iframes to have a threat model similar to a phishing page (see the Threat model part of this explainer below).

## Security and privacy considerations

### Threat model for anonymous iframes

Because anonymous iframes can be embedded in crossOriginIsolated contexts, in browsers without Out-of-Process-Iframes, we have to consider that their embedder can perform a Spectre attack to read any of the anonymous iframes resources, including the HTML. Our approach to this threat is not to prevent the attack from happening, but to avoid loading personalized data so that an attacker has only access to publicly available data.

To do so, we consider a variety of possible attacks:

#### Usage of existing credentials

The most dangerous attack is also the most straightforward. The attacker embeds an anonymous iframe with resources for which the user already has credentials. The attacker then reads the personalized resources inside the iframe, which are not public data.

Anonymous iframes do not have access to existing credentials stored by their origin. This includes cookies. This also includes any data in the origin shared storage, as it could have been retrieved using credentials, hence personalized. Anonymous iframes start from a blank slate to prevent an attacker from loading resources using existing credentials.

#### Usage of new credentials

In this attack, the attacker embeds an anonymous iframe. As explained above, the anonymous iframe starts from a blank slate when it comes to credentials and shared storage. However, the anonymous iframe could register new credentials and use those to request personalized resources. The attacker can then read those. In practice, we can divide this scenario into two. First usage of credentials storing state necessary to make the iframe site work, but which is not particular to a user. This case is not problematic, as it is publicly accessible data. The second one is state personalized by user, which would be acquired after the user logs into the anonymous iframe.

Finally, there is the question of how long such credentials should persist, and how they should be shared across anonymous iframes. For example, one could imagine that the user visits a legitimate page with an anonymous iframe A where they log in. Then they visit an attacker page with an anonymous iframe of the same origin A. If they are still logged into A, the attacker page could steal data from personalized subresources.

If the user is directly typing their credentials in the iframe to log-in, they are logging into the iframe, in a page with a different URL shown in the omnibox. This is similar to a phishing attack both in user action needed for the attack to happen and outcome. This case shouldn’t require extra-mitigation.

When the user is using a popup-driven OAuth flow (or the upcoming WebID API), the situation is harder to understand from the user’s perspective. To prevent this from happening, we restrict the iframe ability to open popups (e.g. all popups are opened with no-opener set), and access the WebID API when it ships.

In terms of lifetime and sharing of credentials, they are bound to the lifetime of the anonymous iframe. So if a page creates an anonymous iframe, documents in the subtree of the anonymous iframe get a clean slate of credentials and shared storage. Documents in the subtree can create credentials. They can share them among themselves (as long as they respect the same-origin policy), and only among themselves. So if two pages are opened at the same time with two anonymous iframes with the same origin, the anonymous iframes cannot share their credentials. Once the anonymous iframe is destroyed, the credentials that were created in its subtree should no longer be accessible. This ensures that we preserve as much functionality for the anonymous iframe as possible, while minimizing the risk of accidentally leaking data.

#### Personalized resources based on network position

This is a variant of the previous attack, where the user embeds an iframe with private subresources the user is only allowed to access due to their network position. For example, resources found on the user private network, or resources personalized based on a user IP address.

We plan on dealing with the private network case by deploying Private Network Access restrictions. During the CORS preflight introduced by [Private Network Access restrictions](https://wicg.github.io/private-network-access/), the servers will be able to check that their resource will be rendered in an anonymous iframe context by checking the Sec-Fetch-COEP header. Note that only local-network documents which enable HTTPS could potentially be exposed (because MIX should prevent HTTP resources from being loaded by a page with COI). This isn't a strong mitigation, but will matter for things like common IoT devices. The other cases of resources personalized based on IP address are arguably a security footgun already. We think that the increased risk there is okay compared to the advantage of not using credentials on more requests overall.

#### Capture of user input

In this attack, the attacker embeds an anonymous iframe that can receive user input. It then reads the user input.

This is not in scope for anonymous iframes. An attacker can already do a phishing attack where they construct a fake page using publicly available resources and trick the user into entering data. This attack is equivalent, since the URL shown to the user in the navigation bar will still be that of the attacker.

#### Anonymous iframes using side-channels to personalize themselves

The anonymous iframe could use side-channels (e.g. broadcast channels, postMessage) to attempt to get a form of personalization despite the lack of credentials. The personalized resources are then readable by the embedder.

Depending on whether the mechanisms highlighted above are a common way of personalizing resources, this might be out-of-scope. What we want to defend against is unsuspecting websites being embedded and attacked by their embedder to steal user data. If the anonymous iframe is bent on escaping the constraints of anonymous iframes to personalize itself, then one can argue that it understands the contexts and risks it is loaded in and accepts them. Provided our security model is safe enough outside of anonymous iframes, the personalization will only affect resources that are same-origin with the iframe anyway. Cross-origin resources to the framed document will still be unpersonalized, making this equivalent to COEP:credentiallesss from a security perspective.

### Privacy

The main privacy threat posed by this API is the risk of a data leak through a side channel attack like Spectre. As detailed in the threat model above, we believe the API provides a meaningful defense against Spectre attacks, and thus does not pose a privacy risk.

## Alternatives considered

### Sandboxed iframe

Sandboxed iframes without the allow-same-origin flag do not have access to storage APIs or cookies for their subresource requests. However, the document of a sandbox iframe can still be requested with credentials, which does not fit the threat model described above. We could change sandboxed iframes so that documents are also requested without credentials.

So why are we proposing introducing a new attribute instead of just using sandboxed iframes with a new sandbox flag?

First, changing the behavior of sandboxed iframes so that their main resource is always requested without credentials could break existing websites, as opposed to introducing a new concept.

Second, we want to minimize the amount of disruption imposed to the content inside the iframe. Using sandboxed iframes means the iframes cannot use cookies or storage APIs at all, nor could they access any other frame in the document. We are worried that this would limit the deployability of the credentialless solution for opting into crossOriginIsolation. We’re looking to provide developers with a solution that is as deployable as possible, which is why we’d rather introduce a new solution that imposes as few restrictions to the iframes as possible.

We could try to codify these restrictions as a sandbox flag, e.g. allow-partitioned-storage. This is probably hard to reconcile with the storage access sandbox flag shipped by Firefox and Safari, especially since a new sandbox flag would be off by default.

This in turn is another issue with relying on sandboxed iframes for COEP deployment. Because all flags are off by default, any new flag could impact the behavior of sandboxed iframes. Not to mention that the syntax is a bit complex due to the need to add every flag but the allow-same-origin to get all functionality but access to cookies/storage.

### Opaque origins

The anonymous iframes model that we propose relies on partitioned storage (see explainer), using a nonce in the storage key. We have also considered attributing opaque origins to the anonymous iframes, similar to sandboxed iframes. This would ensure that the anonymous iframes do not have access to existing credentials and shared storage since their origin has been changed to an opaque one.

This solution runs into compatibility issues:
* To allow anonymous iframes to access one another if they are coming from the same origin we must maintain a mapping of original origin to opaque origin for each anonymous iframe subtree, which is complex.
* We would probably need to standardize what happens when a frame with an opaque origin wants to access a storage API since sandboxed iframes with opaque origins do not have access to storage APIs at all.
* It is not clear how this would interact with other checks pertaining on origin (e.g. X-Frame-Options, various CSP checks, …) potentially leading to further breakages.

## Self-Review Questionnaire: Security and Privacy

#### What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

The `Sec-Fetch-COEP` header exposes the COEP of the environment a resource will be rendered in. This allows a server to decline answering a request if they do not want their resource to be embedded in a more dangerous environment. The `window.anonymous` method exposes whether a document is loaded in an anonymous iframe or not, allowing a document to change its behavior depending on the availability of existing credentials or stored resources.

#### Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes. The only thing exposed is whether a frame is anonymous or not.

#### How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

The feature does not deal with PII.

#### How do the features in your specification deal with sensitive information?

The feature does not deal with sensitive information.

#### Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No the feature does not introduce new state for an origin.

#### Do the features in your specification expose information about the underlying platform to origins?

No the feature behaves the same regardless of the underlying platform.

#### Does this specification allow an origin to send data to the underlying platform?

No this feature does not change what data an origin is allowed to send to the underlying platform.

#### Do features in this specification allow an origin access to sensors on a user’s device?

No the feature has no impacty on sensor access.

#### What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

The `Sec-Fetch-COEP` header exposes the COEP of the environment a resource will be rendered in. This corresponds to the COEP of the embedder of the resource, or the fact that a resource is loading in an anonymous iframe. This allows a server to decline answering a request if they do not want their resource to be embedded in a more dangerous environment.

#### Do features in this specification enable new script execution/loading mechanisms?

No the feature does not enable new script execution/loading.

#### Do features in this specification allow an origin to access other devices?

No the feature is strictly confined to one device.

#### Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No the feature has no impact on UI.

#### What temporary identifiers do the feautures in this specification create or expose to the web?

No temporary identifiers are created.

#### How does this specification distinguish between behavior in first-party and third-party contexts?

There is no distinction between first-party and third-party ability to embed anonymous iframes. This is due to the recursive nature of COEP. To deploy COEP in a frame, all child frames need to deploy COEP first. Since this is a mechanism to help with the deployment of COEP, we want to offer third-party iframes the option to ease their own COEP deployment by being able to embed their own third party content as anonymous iframes.

#### How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

No difference with regular mode.

#### Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes.

#### Do features in your specification enable origins to downgrade default security protections?

This feature has no impact on secure context and same-origin policy. It does allow to use features which are right now gated behind having COOP and COEP enabled. However, it imposes restrictions on documents to make this safe.
   
</details>

