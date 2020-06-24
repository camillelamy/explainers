# Side channel threat model for Web APIs

## Overview

This document aims to provide a threat model for side channel leaks due to Web APIs. Our goal is to create clear guidelines to understand the security threat posed by various APIs. These guidelines will take into account the surface of attack and the information exposed by a particular API to make a recommandation on having this API use appropriate mitigation.

## Background

Embedding third-party content is the core of the Web Platform. However, this presents a security risk. The browser requested third-party content for a webpage using credentials set by the third-party content. If the first party page could read the third-party content it embeds, it can get private information the user does not want to see shared. This is why we introduced the [same-origin policy](https://web.dev/same-origin-policy/) to restrict how scripts interact with content coming from another origin. In particular, same-origin policy allows to embed cross-origin content. However, it blocks read access to cross-origin content.

Enters Specter. Specter showed that same-origin policy was not enough to protect resources from third-party attacker scripts. In the [Specter threat model](https://chromium.googlesource.com/chromium/src/+/master/docs/security/side-channel-threat-model.md), without mitigation, embedding a cross-origin resource is equivalent to gaining read access to it. Even if browsers enforce the same-origin policy, a malicious actor can exploit the Specter vulnerability to read all of the resources present in its adress space.

To exploit a Specter vulnerability, a bad actor needs to have access to precise resolution timers. They can gan gain access two these in two ways, First by exploiting the browser and using native timers, which is beyond the scoped of this document is concerned about. Second, by using Web APIs that expose precise resolution timers, such as [SharedArrayBuffers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer). This is the kind of threat we are concerned with here.

While SharedArrayBuffers used in conjunction with Specter is the most striking example of using a WebAPI to gain information about third-party resources bypassing the same origin policy, it is not the only threat. As we work on delivering more powerful APIs for the WebPlatform, in particular around performance measurement like the [memory measurement API](https://github.com/WICG/performance-measure-memory), the surface of attacks also grows. We cannot have those powerful APIs and only rely on same-origin policy to ensure that Web APIs will not leak information from one origin to another.

We have been working on mitigations. For Specter itself, browsers have worked on delivering [Site Isolation](https://security.googleblog.com/2018/07/mitigating-spectre-with-site-isolation.html). Site Isolation ensures that documents only share their adress space with same-site documents. This effectively mitigates the risk of one document attacking another document that shares the same process, using either browser exploit or WebAPIs. However, it
is not enough to mitigate the threat coming from Web APIs side channels. First, it does not ship on all platforms and browsers, meaning that potentially risky APIs like SharedArrayBuffers are still blocked in many browsers or on some platforms. Second, it does not prevent attacks hapenning inside a document. Third, it is not web facing, so web pages cannot know if a browser has Site Isolation enabled or not.

To address those issues, we have been working on two web facing APIs, COOP and COEP, that puts the page in a [crossOriginIsolated](https://web.dev/coop-coep/) state. This state essentially reverses the model of embedable third-party content by default + mitigation using the same-origin policy. When the page is crossOriginIsolated, third-party content can only be embedded if it opts into being embedded cross-origin. However, from a threat model perspective, the same-origin policy is no longer applied. Because crossOriginIsolated enables powerful APIs like SharedArrayBuffers that can be used in a Specter exploit, all scripts in a crossOriginIsolated context have read access to resources in their document, regardless of origin.

COOP and COEP comes in addition to other kind of mitigations that regulates the information exposed by WebAPIs, for example [timing-allow-origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Timing-Allow-Origin) and [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS). We do have ways to mitigate the threat WebAPIs could pause. We need to make sure we properly assess new (and existing) web APIs for risk, and restrict them beyond the appropriate mitigation when needed. We might also consider whether additional forms of mitigation are needed.

## Threat model

We are concerned about a script gaining information about another cross-origin resource by using a WebAPI. To mitigate this risk, we need to assess the risk each Web API pauses. To better understand the risk, we should look at two things:

1. What kind of information does the Web API exposes?
2. What surface of attack does the Web API enables?

Let's look at both in more details.

### Information exposed

#### Read access

In this case, the web API gives a script read-access to another resource. Taking into account Specter, any Web API that exposes a precise timer essentially gives read access to all other resources on the document. In terms of threat, this allows an attacker to read the result of any third-party credentialed request.

#### Disambiguation between responses

In this case, the web API allows a script to disambiguate between two possible results of a third-party credentialed request even though it cannot read the response. For example, having access to a response size can allow to differentiate between two different responses to the same request. In terms of threat, an attacker can make deuctions on the user credentials based on the reponse that is sent. For example, imagine a request results in a different response depending on whether the user is logged in or not. An attacker can make the same request and determine the logged in status of the user if they can differentiate between the reponse that would be sent had the user been logged or the one if they hadn't.

Trying to see if an API fits into this category is harder than the previous one. In particular, many APIs will only have a probability to differentiate between two reponses. The probability should be high enough for the threat to be effective. To get back to an example, using resource size to differentiate between two responses is very likely to work as an attack, as it is very unlikely that the two will have the exact same size. Using frame animation timing to infer the size of an image based on frame rendering time and then use this size to disambiguate between responses is unlikely to work reliably as an attack.

#### Metadata about a resource

In this case, the API exposes metadata that is not necessarily related to the content of the resource itself. For example, information related to how the resource was fetched (protocol, server timings, etc...). The threat here is that a server might not want to see this kind of information exposed cross-origin.

#### None of the above

In that case, the Web API does not present a threat of side channel attack. Great! In all other cases, though, access to the Web API needs to be gated behind appropriate mitigation.

### Surface of attack

#### Single resource

#### Page

#### Browsing context group


## Mitigations
