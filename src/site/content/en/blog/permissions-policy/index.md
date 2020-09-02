---
title: Introduction to feature policy
subhead:
  Modify API behaviors using headers and meta tags
authors:
  - ericbidelman
date: 2018-06-26
updated: 2020-10-15
# hero: hero.jpg
# alt: TBD

description: |
  Feature Policy allows developers to selectively enable, disable, and modify the behavior of certain APIs and features in the browser. It's like CSP, but for features. Shipped in Chrome 60.
tags:
  - blog
  - ux
  - chrome60
  - chrome74
  - feature-policy
---

[Feature Policy][spec] allows you to selectively enable, disable, and
modify the behavior of certain APIs and web features in the browser. __It's like
[CSP](/web/fundamentals/security/csp/) but instead of controlling security, it
controls features.__

The feature policies themselves are little opt-in agreements between developer
and browser that can help foster the goal of building (and maintaining) high
quality web apps.

## Background

Building for the web is a rocky adventure. It's hard enough to build a top-notch
web app that nails performance and uses all the latest best practices. It's even
harder to keep that experience great over time. As your project evolves,
developers come on board, new features land, and the codebase grows. That
Great Experience &trade; you once achieved may begin to deteriorate and UX
starts to suffer. Feature Policy is designed to keep you on track.

With Feature Policy, you opt in to a set of "policies" for the browser to
enforce on specific features used throughout your site. These policies restrict
what APIs the site can access or modify the browser's default behavior for
certain features.

Here are examples of things you can do with Feature Policy:

- Change the [default behavior](/web/updates/2017/09/autoplay-policy-changes#iframe)
  of `autoplay` on mobile and third-party videos.
- Restrict a site from using sensitive APIs like `camera` or `microphone`.
- Allow iframes to use the `fullscreen` API.
- Block the use of outdated APIs like synchronous XHR and `document.write()`.
- Ensure images are sized properly (e.g. prevent layout thrashing) and that they
  are not too big for the viewport (e.g. waste user's bandwidth).

**Policies are a contract between developer and browser**. They inform the
browser of the developer's intent and thus, help keep us honest when our app
tries to go off the rails and do something bad. If the site or embedded
third-party content attempts to violate any of the developer's preselected
rules, the browser overrides the behavior with better UX or blocks the API
altogether.

## Using Feature Policy

Feature Policy provides two ways to control features:

* Through the `Feature-Policy` HTTP header.
* With the `allow` attribute on iframes.

{% Aside 'key-term' %}
  The biggest difference between the HTTP header and the `allow` attribute is
  that the `allow` attribute only controls features within an iframe. The header
  can control features in the main response and any iframe-contained content
  within the page. This is because [iframes inherit policies from their parent
  pages](#inheritancerules).
{% endAside %}

### The Feature-Policy HTTP header

The easiest way to use Feature Policy is by sending the [`Feature-Policy`][header] HTTP
header with the response of a page. This header sets a policy or policies that
the browser must respect for a given origin:

```text
Feature-Policy: <feature> <allowlist>
```

The supported features is extensive and varies by browser. You can find a
[support list on MDN][header-compat]. The allow list can take
several values:

- `*`: The feature is allowed in top-level browsing contexts and in nested
  browsing contexts (iframes).
- `'self'`: The feature is allowed in top-level browsing contexts and
  same-origin nested browsing contexts. It is disallowed in cross-origin
  documents in nested browsing contexts.
- `'none'`: The feature is disallowed in top-level browsing contexts and
disallowed in nested browsing contexts.
- `<origin(s)>`: The feature is allowed in the specified origins (e.g.
  `https://example.com`).

**Example**

Let's say you want to block all content from using the Geolocation API across
your site. You can do that by sending a strict allowlist of `'none'` for the
`geolocation` feature:

```text
Feature-Policy: geolocation 'none'
```

If a piece of code or iframe tries to use the Geolocation API, the browser
blocks it. **This is true even if the user has previously given
permission to share their location**.

<figure class="w-figure">
  <img src="./violation.png"
       class="w-screenshot" alt="Violating the set geolocation policy."
       title="Violating the set geolocation policy.">
  <figcaption>Violating the set geolocation policy.</figcaption>
</figure>

In other cases, it might make sense to relax this policy a bit. You can allow
your own origin to use the Geolocation API but prevent third-party content from
accessing it by setting `'self'` in the allow list:

```text
Feature-Policy: geolocation 'self'
```


### The iframe `allow` attribute

The second way to use Feature Policy is for controlling content within
an `iframe`. Use the `allow` attribute to specify a policy list for
embedded content:

```html
<!-- Allow all browsing contexts within this iframe to use fullscreen. -->
<iframe src="https://example.com..." allow="fullscreen"></iframe>

<!-- Equivalent to: -->
<iframe src="https://example.com..." allow="fullscreen *"></iframe>

<!-- Allow only iframe content on a particular origin to access the user's location. -->
<iframe src="https://another-example.com/demos/..."
        allow="geolocation https://another-example.com"></iframe>
```

{% Aside %}
  Frames inherit the policy settings of their parent page. If the page and
  iframe both specify a policy list, the more restrictive list is used. For more
  information, see [Inheritance rules](#inheritancerules).
{% endAside %}

#### What about the existing iframe attributes?

Some of the [features controlled by Feature Policy](#list) have an existing
attribute to control their behavior. For example, `<iframe allowfullscreen>`
allows iframe content to use the `HTMLElement.requestFullscreen()` API. There's
also the `allowpaymentrequest` and `allowusermedia` attributes for allowing the
[Payment Request API](/web/fundamentals/payments/) and `getUserMedia()`,
respectively.

Try to **use the `allow` attribute instead of these old attributes** where
possible. For cases where you need to support backwards compatibility using the
`allow` attribute with an equivalent legacy attribute is acceptable (e.g.
`<iframe allowfullscreen allow="fullscreen">`). Just note that where rules
conflict, the more restrictive policy wins. For example, the following iframe
would not be allowed to enter fullscreen because `allow="fullscreen 'none'"` is
more restrictive than `allowfullscreen`:

```html
<!-- Blocks fullscreen access if the browser supports feature policy. -->
<iframe allowfullscreen allow="fullscreen 'none'" src="&hellip;">
```

## Controlling multiple policies at once

You can control several features at once with one HTTP header by separating the
policy directives with a semicolon (`;`):

```text
Feature-Policy: unsized-media 'none'; geolocation 'self' https://example.com; camera *;
```

or by sending a separate header for each policy:

```text
Feature-Policy: unsized-media 'none'
Feature-Policy: geolocation 'self' https://example.com
Feature-Policy: camera *;
```

This example does the following:

- Disallows the use of `unsized-media` for all browsing contexts.
- Disallows the use of `geolocation` for all browsing contexts except for the
  page's own origin and `https://example.com`.
- Allows `camera` access for all browsing contexts.

To set multiple policies on an iframe:

```html
<!-- Blocks the iframe from using the camera and microphone
     (if the browser supports feature policy). -->
<iframe allow="camera 'none'; microphone 'none'">
```

## JavaScript Interface

In addition to the `Feature-Policy` HTTP header, there's also a
[FeaturePolicy][jsapi] interface available in JavaScript. As with the HTTP
header, [support varies by browser][jsapi-compat]. Among the capabilities of
this AOU us a way to deterimine which features are allowed by a page, frame or
browser. To get an instance of the interface:

* In Chromium forks, call `document.featurePolicy` or `frame.featurePolicy`.
* In Firefox, call `document.policy` or `frame.policy`.

### Example

The example below illustrates the results of sending a policy of
`Feature-Policy: geolocation 'self'` on the site `https://example.com`:

```js
/* @return {Array<string>} List of feature policies allowed by the page. */
document.featurePolicy.allowedFeatures();
// → ["geolocation", "midi",  "camera", "usb", "autoplay",...]

/* @return {boolean} True if the page allows the 'geolocation' feature. */
document.featurePolicy.allowsFeature('geolocation');
// → true

/* @return {boolean} True if the provided origin allows the 'geolocation' feature. */
document.featurePolicy.allowsFeature('geolocation', 'https://another-example.com/');
// → false

/* @return {Array<string>} List of feature policies allowed by the browser
regardless of whether they are in force. */
document.featurePolicy.features();
// → ["geolocation", "midi",  "camera", "usb", "autoplay",...]

/* @return {Array<string>} List of origins (used throughout the page) that are
   allowed to use the 'geolocation' feature. */
document.featurePolicy.getAllowlistForFeature('geolocation');
// → ["https://example.com"]
```

## List of policies

So what features can be controlled through Feature Policy? The list is already extensive. Information is available through the browser support tables linked to above. The list continues to grow as different browsers adopt the spec.

Right now, there's a lack of documentation on what policies are implemented and
how to use them. The list will also grow over time as different browsers adopt
the spec and implement various policies. Feature policy will be a moving target
and good reference docs will definitely be needed. There are other ways to learn
what's available.

- Check out our [Feature Policy Kitchen Sink][sink] of demos. It has examples
  of many policies implemented in Blink.
- Check [chromestatus.com][chromestatusfilter] for the policies that have been
  implemented or are being considered in Blink.
- In your developer tools console open `about:blank` and call
  `document.featurePolicy.allowedFeatures()` (Chrome) or
  `document.Policy.allowedFeatures()` (Firefox).

While many of the policies are [documented on MDN][header], you can find
explainers for newer policies in the [spec's GitHub
repo](https://github.com/WICG/feature-policy/tree/master/policies).
## FAQ

### When do I use Feature Policy?

All policies are opt-in, so use Feature Policy when and where it makes sense.
For example, if your app is an image gallery, the `maximum-downscaling-image`
policy would help you avoid sending gigantic images to mobile viewports.

Other policies like `document-write` and `sync-xhr` should be used with more
care. Turning them on could break third-party content like ads. On the
other hand, **Feature Policy can be a gut check** to make sure your pages
never uses these terrible APIs.

{% Aside %}
  Enable Feature Policy on your own content before enabling it on third-party
  content.
{% endAside %}

### Do I use Feature Policy in development or production?

Both. We recommend turning policies on during development and keeping the
policies active while in production. Turning policies on during development can
help you start on the right track. It'll help you catch unexpected
regressions before they happen. Keep policies turned on in production
to guarantee a certain user experience for users.

### Is there a way to report policy violations to my server?

A [Reporting API](https://github.com/WICG/feature-policy/blob/master/reporting.md)
is [in the works](https://github.com/WICG/feature-policy/blob/master/reporting.md)!
Similar to how sites can opt-in to receiving reports about
[CSP violations](https://www.chromestatus.com/feature/5826576096690176) or
[deprecations](https://www.chromestatus.com/feature/4691191559880704), you'll
be able to receive reports about feature policy violations in the wild.

### What are the inheritance rules for iframe content?

Scripts (either first or third-party) inherit the policy of their browsing
context. That means that top-level scripts inherit the main document's policies.

`iframe`s inherit the policies of their parent page. If the `iframe` has an
`allow` attribute, the stricter policy between the parent page and the `allow`
list, wins. For more information on `iframe` usage, see the
[`allow` attribute on iframes](#iframe).

Disabling a feature policy is a one-way toggle. Once a policy is disabled, it
cannot be re-enabled by any frame or descendant.
{: .caution }

### If I apply a policy, does it last across page navigations?

No. The lifetime of a policy is for a single page navigation response. If
the user navigates to a new page, the `Feature-Policy` header must be explicitly
sent in the new response for the policy to apply.

### What browsers support Feature Policy?

[See caniuse.com](https://caniuse.com/#feat=feature-policy) for the latest
details on browser support.

As of now, Chrome is the only browser to support feature policy. However,
since the entire API surface is opt-in or feature-detectable, **Feature Policy
lends itself nicely to progressive enhancement**.

## Conclusion

Feature Policy can help provide a well-lit path towards better UX and
good performance. It's especially handy when developing or maintaining an app
since it can help avoid potential footguns before they sneak into your codebase.

**Additional resources**:

- [Feature Policy Explainer][explainer]
- [Feature Policy spec][spec]
- [Kitchen Sink Demos][sink]
- [Feature Policy DevTools Extension](https://chrome.google.com/webstore/detail/feature-policy-tester-dev/pchamnkhkeokbpahnocjaeednpbpacop) - Tester for trying out feature policies on a page.
- [chromestatus.com entries][chromestatusfilter]

[spec]: https://wicg.github.io/webappsec-feature-policy/
[header]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Feature-Policy
[header-compat]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Feature-Policy#Browser_compatibility
[jsapi]: https://developer.mozilla.org/en-US/docs/Web/API/FeaturePolicy
[jsapi-compat]: https://developer.mozilla.org/en-US/docs/Web/API/FeaturePolicy#Browser_compatibility
[chromestatus]: https://www.chromestatus.com/features/5694225681219584
[chromestatusfilter]: https://www.chromestatus.com/features#component%3A%20Blink%3EFeaturePolicy
[sink]: https://feature-policy-demos.appspot.com/
[explainer]: https://docs.google.com/document/d/1k0Ua-ZWlM_PsFCFdLMa8kaVTo32PeNZ4G7FFHqpFx4E/

