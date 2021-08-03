# Make CrossOriginIsolation easily deployable
clamy@google.com

## SharedArrayBuffers, Spectre and CrossOriginIsolation

The Web relies on the same-origin security model. Following this security model, documents can embed cross-origin resources such as images, scripts, or iframes. However, they do not have access to the actual bits of the resources they embed.

Following Spectre, this is no longer true. Spectre allows an attacker to read any resource located in the same process. Browsers put all the subresources a document uses in the same process as the document. Same-site documents in a browsing context group are also always located in the same process. Finally, in browsers without Site Isolation, all documents of the same page must be located in the same process. This means that any popup opened by a page must be placed in the same process (because it could eventually embed an iframe that is same-site with a document in the page that opened it). This means that the attack surface of Spectre is:
* With SiteIsolation: any resource located in a same-site document in the browsing context group.
* Without SiteIsolation: any resource in the browsing context group.

Spectre efficiency is correlated on the precision of timers used in the attack. Following the discovery of the attack, the WebPlatform took steps to reduce the precision of timers. In particular, this led to restrictions on SharedArrayBuffers as SharedArrayBuffers can be used to construct high precision timers. Browsers without SiteIsolation decided to restrict them in order to mitigate the risks of a Spectre attack. Chrome desktop, which had full Site Isolation, continued shipping SharedArrayBuffers, including their passing.

This was a suboptimal situation. First, even with SiteIsolation, regular subresources are not protected from a Spectre attack. Second, implementing Site Isolation is a heavy engineering investment for browsers, and may not even be possible on all platforms (in particular on low-end mobile devices). SharedArrayBuffers provide a shared memory API that is particularly useful for heavy web apps that perform a lot of computations. So there was a desire to bring SharedArrayBuffers on all browsers, without waiting for SiteIsolation support.

In order to do so, CrossOriginOpenerPolicy, CrossOriginEmbedderPolicy and the concept of crossOriginIsolation were designed. Rather than preventing a Spectre attack from happening, COOP and COEP ensures that every subresource in the Spectre attack surface (the browsing context group) agrees to be loaded in a potentially dangerous environment where they could be legible by a cross-origin resource. We name such environments crossOriginIsolated.

The core of the mechanism is Cross-origin Embedder Policy (COEP). When a document enables COEP, it can only load cross-origin subresources through CORS or if they have an appropriate CORP header. The CORP header serves as an opt-in, where the resource signifies that it is ok to be loaded in the crossOriginIsolated environment.

COEP also restricted the load of iframes. Documents embedded in a COEP document also have to enable COEP. And cross-origin iframes embedded in a COEP document must also send an appropriate CORP header or they will be blocked. Overall, this recursive mechanism ensures that if a top-level frame sets COEP, every document in the page will also have set COEP. In turn, this ensures that every resource on the page opted into being loaded in a risky environment (either through a CORP header or because they were loaded in a same-origin COEP document).

Cross-origin Opener Policy (COOP) can be used to ensure that all top-level documents in a browsing context group have the same origin and set COEP. This happens when a top-level document sets COOP same-origin and COEP require-corp. COOP will then ensure that any navigation to/from a page with a different origin or a different COOP will result in a browsing context group switch. Similarly, any popup opened to a document with a different origin or a different COOP will be placed in a different browsing context group. Because COEP is included in the special COOP value same-origin-plus-coep, this ensures that all top-level pages set COEP.

Since all top-level pages set COEP, then all documents in the browsing context group also set COEP. As explained above, this means that every subresource of each page in the browsing context group has opted into being loaded in a risky environment. This is not true for the top-level document of each page. We do not check CORP for a top-level navigation to a COEP page. Yet the top-level document could be at risk from cross-origin pages in the browsing context group. This is why COOP also ensures that all top-level documents in a crossOriginIsolated browsing context group are same-origin with each other. With this additional restriction, we can truly assert that every subresource in the Spectre attack surface (the browsing context group) agrees to be loaded in a crossOriginIsolated environment.

COOP, COEP and crossOriginIsolation now ship on Firefox and Chrome. They have allowed to enable SharedArrayBuffers on Firefox and Chrome for Android. However, SharedArrayBuffers were already shipping on Chrome for desktop without the crossOriginIsolated restriction. Chrome desktop is also planning to add the crossOriginIsolated to SharedArrayBuffer usage, in order to align with Firefox and Chrome Android. It will also improve security for pages that use SharedArrayBuffers, as crossOriginIsolation protects subresources against Spectre attacks while SiteIsolation acts at the frame level.

SharedArrayBuffers were already being used by ~2% of pages on Chrome desktop, which now need to deploy crossOriginIsolation or might eventually stop functioning. At the same time, several heavy web applications (e.g. spreadsheet application, drawing application) are also interested in deploying crossOriginIsolation in order to gain access to SharedArrayBuffers and improve their performance.

## Adoption Challenges

As websites have been trying to deploy crossOriginIsolation to keep their applications that use SharedArrayBuffers working, or to gain access to SharedArrayBuffers, we have identified 3 main challenges with the deployment of COOP and COEP:
* OAuth/payment flows using popups
* 3rd party subresources not having CORP headers
* Embedding of legacy/arbitrary 3rd party iframes.

### Popup flows

COOP same-origin is required to enable crossOriginIsolation. COOP same-origin forces all cross-origin popups to be put in another browsing context group. This means that the popup does not have access to its opener and cannot use PostMessage with it. Similarly, the crossOriginIsolated page cannot access the popup it opens.

This is problematic when it comes to some OAuth and payment flows, which can be implemented through popups. For example, the [PayPal demo page](https://developer.paypal.com/docs/checkout/) is showing a popup based payment flow.

So a page cannot use a popup-based OAuth or payment flow and SharedArrayBuffers with the way crossOriginIsolation is designed right now.

### 3rd party subresources

Many websites rely on loading subresources such as images from cross-origin sources like CDNs. Cross-origin isolation requires websites set COEP require-corp. COEP require-corp will block non-CORS requests to cross-origin subresources, unless the cross-origin subresource has a CORP cross-origin header. So websites that wish to deploy COEP must wait until all the third party properties from which they embed subresources send CORP headers with their resources. In some cases, the resources might no longer be properly maintained anymore, meaning their owners might never send the required CORP headers with them.

### 3rd party iframes

The problem of embedding 3d party resources is even worse when it comes to 3rd party iframes. COEP require-corp will only allow embedding iframes if they also have a COEP require-corp header and a CORP cross-origin header. This requirement propagates to any child frame of the 3d party iframe.

Several websites rely on embedding arbitrary 3rd party iframes, and may not expect them to support COEP any time soon. For example, an ad provider cannot make assumptions on the COEP support of the ad being displayed in an ad iframe. Meaning that right now, ads such as Google Adwords cannot support COEP. In turn, pages that rely on this kind of ads for their business model cannot deploy COEP and crossOriginIsolation.

Similarly, websites may also embed legacy 3rd party iframes. For example, Google Earth allows users to create balloons where they can register their own HTML content that will be displayed in an iframe. It is highly unlikely that users will update their content to support COEP. Meaning Google Earth either has to stop using SharedArrayBuffers or stop displaying user-created content.

## Proposed solutions

All those challenges have prevented websites from being able to deploy crossOriginIsolation as they cannot currently do so without breaking existing functionality on their sites. This means that they  cannot gain access to SharedArrayBuffers, impairing their performance. Or even worse, for existing users on Chrome for desktop they risk losing access to SharedArrayBuffers, which might break their sites.

To fix this, we are planning three extensions to COOP and COEP that address the challenges presented above. With these three extensions, websites should be able to support crossOriginIsolation without breaking their apps and be able to use SharedArrayBuffers.

The three planned extensions are:
* COOP same-origin-allow-popups-plus-coep
* COEP credentialless
* Anonymous iframes

### COOP same-origin-allow-popups-plus-coep

To allow crossOriginIsolated pages to use popup-based OAuth/payment flows, we plan to have COOP same-origin-allow-popups enable crossOriginIsolation when used in conjunction with COEP. This would create a new COOP value: COOP same-origin-allow-popups-plus-coep. Like COOP same-origin-allow-popups, COOP same-origin-allow-popups-plus-coep allows popups opened by the page to stay in the same browsing context group. The popups can still communicate with their opener via WindowProxy postMessage.

Without further restriction, this would be problematic from a security perspective as the attack surface of Spectre is the browsing context group in browsers without SiteIsolation. We plan to add additional restrictions to COOP same-origin-allow-popups-plus-coep can be placed in a separate process from non crossOriginIsolated popups it opens, even on browsers without SiteIsolation. This then reduces the surface of attack of a Spectre attack to the crossOriginIsolated page, making it safe to have popups in the same browsing context group.

### COEP credentialless

To simplify the deployment of COEP on pages that embed 3rd party subresources, we plan to introduce a new COEP mode: COEP credentialless.

COEP require-corp will block non-CORS cross-origin requests unless they have a CORP header. This ensures that cross-origin resources have opted into being loaded in the crossOriginIsolated environment.

COEP credentialless takes a different approach. Instead of ensuring that non-CORS cross-origin subresources opt into being loaded into the dangerous environment, we ensure that these resources will not be of interest to an attacker. To do this, we stop sending credentials with non-CORS cross-origin requests. In conjunction with Private Network access, this should ensure that an attacker only has access to resources that it could already request by themselves.

### Anonymous iframes

Anonymous iframes are a generalization of COEP credentialless to support 3rd party iframes that may not deploy COEP. Like with COEP credentials, we replace the opt-in of cross-origin subresources by avoiding to load non-public resources.

Because this applies to iframes, the restrictions are more numerous than in COEP credentialless. In particular, the documents loaded in anonymous iframes have not opted into being loaded in a crossOriginIsolated environment. In COEP regular mode, we treat the opt-in of the document as an opt-in for all its same-origin subresources. This no longer applies, so anonymous iframes are putting restrictions on credentials for all requests (cross-origin and same-origin). In addition, we’re also restricting the access to storage APIs for documents in anonymous iframes, as they could be used to load personalized data.

