# Anonymous iframes

## A problem

Sites that wish to continue using SharedArrayBuffer must opt-into cross-origin isolation. Among other things, cross-origin isolation will block the use of cross-origin resources and documents unless those resources opt-into inclusion via either CORS or CORP. This behavior ships today in Firefox, and Chrome aims to ship it as well in Chrome 92.

CrossOriginIsolation is also being used to gate new features, including features that require additional opt-in in addition to crossOriginIsolation, like [getViewportMedia](https://github.com/w3c/mediacapture-screen-share/issues/155#issuecomment-812269225).

The opt-in requirement is generally positive, as it ensures that developers have the opportunity to adequately evaluate the rewards of being included cross-site against the risks of potential data leakage via those environments. It poses adoption challenges, however, as it does require developers to adjust their servers to send an explicit opt-in. This is challenging in cases where there's not a single developer involved, but many. Google Ads, for example, includes third-party content, and it seems somewhat unlikely that they'll be able to ensure that all the ads creators will do the work to opt-into being loadable.

It seems clear that adoption of any opt-in mechanism is going to be limited. From a deployment perspective (especially with an eye towards changing default behaviors), it would be ideal if we could find an approach that provided robust-enough protection against accidental cross-process leakage without requiring an explicit opt-in. We presented a first version of this mechanism in [Credentiallessness](https://github.com/mikewest/credentiallessness), which evolved into COEP credentialless. This new mode of COEP tackles subresources included in a frame. It is a good solution for deploying COEP when one includes cross-origin subresources without CORP headers, that come from a CDN for example. However, we still need to ease deployment of COEP for pages that embed cross-origin frames.

## A proposal

COEP, and the opt-in mechanism for [getViewportMedia](https://github.com/w3c/mediacapture-screen-share/issues/155#issuecomment-812269225) currently tackles data leak attacks by ensuring that cross-origin resources explicitly opt into being loaded in an environment with higher risks. This way, servers can protect vulnerable resources by not having them opt into being loaded in high risk environments. COEP cors-or-credentialles and the mechanism we are proposing, anonymous iframes, take a different approach. Instead of requiring opt-in to protect vulnerable resources, we aim to only make requests that result in public data being loaded in high-risk environments. Essentially, this means preventing personalized resources from being loaded.

### What are anonymous iframes?

Documents can create anonymous iframes by adding an attribute to the iframe tag:

`<iframe src=”example.com” credentials=omit>`

This property is stored on the Browsing Context. It is inherited by any child browsing context of the anonymous iframe, e.g.:

![](/anonymous_iframe1.png)

Similar to sandbox flags, the attribute can be changed programmatically, but the change will only take effect on the next navigation taking place inside the iframe.

In parallel with the iframe attribute, we plan to add a new Fetch Metadata header to outgoing requests to inform servers of the COEP/anonymous context in which the response will be embedded:

* `Sec-Fetch-COEP: none`: the request will not be rendered in a COEP context, either because this is a top-level navigation or the frame in which the resource will be rendered in a context with a COEP of unsafe-none.
* `Sec-Fetch-COEP: require-corp`: the resource will be rendered in a context with a COEP of require-corp.
* `Sec-Fetch-COEP: credentialless`: the resource will be rendered in a context with a COEP of credentialless.
* `Sec-Fetch-COEP: anonymous`: the resource will be rendered in an anonymous iframe.

Additionally, we plan on adding a `credentials` method on the `Document` object. By default, this will return `present`. In anonymous iframes it will return `omit`, allowing a document to check whether it was loaded in an anonymous iframe.

### Anonymous iframes and credentials

Anonymous iframes cannot use existing credentials and shared storage for their origin. They are given a blank slate. Unlike sandboxed frames, they can use storage APIs and register cookies. However, those credentials and storage can only be shared by documents in anonymous iframes in the page (provided they meet origin restrictions). They will no longer be accessible once the page has navigated. Essentially, anonymous iframes are given a temporary [storage shelf](https://storage.spec.whatwg.org/#storage-shelf) partitioned to anonymous iframes in the page.

To achieve this, we rely on modifying the [storage key](https://storage.spec.whatwg.org/#storage-key) used to access shared storage by anonymous iframes. As part of the [client-side storage partitioning effort](https://privacycg.github.io/storage-partitioning/) (declined across [storage APIs](https://github.com/wanderview/quota-storage-partitioning/blob/main/explainer.md), [network state](https://github.com/MattMenke2/Explainer---Partition-Network-State) and [Cookie State](https://github.com/DCtheTall/CHIPS)), the storage key of an environment will no longer be its simple origin as currently described in the spec. Instead it will be a combination of the origin and the top-level site URL. In an anonymous iframe, we will replace the top-level site URL in the partition key by a nonce value, determined once per page. This nonce will be recomputed every time the top-level frame navigates. This ensures that anonymous iframes cannot share storage keys with non-anonymous iframes. Because the nonce is changed on every page navigation, anonymous iframes do not share storage across different pages, or across navigations.

![](/anonymous_iframe2.png)

*Storage and credentials are only shared among anonymous iframes, following normal site/origin access checks.*

![](/anonymous_iframe3.png)

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

We plan on dealing with the private network case by deploying Private Network Access restrictions. During the CORS preflight introduced by [Private Network Access restrictions](https://wicg.github.io/private-network-access/), the servers will be able to check that their resource will be rendered in an anonymous iframe context by checking the Sec-Fetch-COEP header. Note that only local-network documents which enable HTTPS could potentially be exposed (because MIX should prevent them from being loaded by a page with COI). This isn't a strong mitigation, but will matter for things like common IoT devices. The other cases of resources personalized based on IP address are arguably a security footgun already. We think that the increased risk there is okay compared to the advantage of not using credentials on more requests overall.

#### Capture of user input

In this attack, the attacker embeds an anonymous iframe that can receive user input. It then reads the user input.

This is not in scope for anonymous iframes. An attacker can already do a phishing attack where they construct a fake page using publicly available resources and trick the user into entering data. This attack is equivalent, since the URL shown to the user in the navigation bar will still be that of the attacker.

#### Anonymous iframes using side-channels to personalize themselves

The anonymous iframe could use side-channels (e.g. broadcast channels, postMessage) to attempt to get a form of personalization despite the lack of credentials. The personalized resources are then readable by the embedder.

Depending on whether the mechanisms highlighted above are a common way of personalizing resources, this might be out-of-scope. What we want to defend against is unsuspecting websites being embedded and attacked by their embedder to steal user data. If the anonymous iframe is bent on escaping the constraints of anonymous iframes to personalize itself, then one can argue that it understands the contexts and risks it is loaded in and accepts them. Provided our security model is safe enough outside of anonymous iframes, the personalization will only affect resources that are same-origin with the iframe anyway. Cross-origin resources to the framed document will still be unpersonalized, making this equivalent to COEP cors-or-credentialless from a security perspective.

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
