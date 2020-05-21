---
title: Know your code health
subhead: Find deprecated APIs in your production apps
authors:
  - ericbidelman
date: 2019-08-21
updated: 2020-05-27
hero: hero.jpg
alt: A reporter observing something.
description: |
  `ReportingObserver` lets you know when your site uses a deprecated API or runs
  into a browser intervention. The basic functionality originally landed in Chrome
  69. As of Chrome 84, it can be used in workers. It's pretty simple.
tags:
  - blog # blog is a required tag for the article to show up in the blog.
---

`ReportingObserver` lets you know when your site uses a deprecated API or runs
into a [browser intervention][interventions]. The basic functionality originally
landed in Chrome 69. As of Chrome 84, it can be used in workers. It's pretty
simple to use as you can see from the code sample below:

```js
const observer = new ReportingObserver((reports, observer) => {
  for (const report of reports) {
    console.log(report.type, report.url, report.body);
  }
}, {buffered: true});

observer.observe();
```

Use the callback to send reports to a backend or analytics provider for
analysis.

Why is that useful? Until this API, deprecation and intervention warnings were
only available in the DevTools as console messages. Interventions, in
particular, are only triggered by various real-world constraints like device and
network conditions. Thus, you may never even see these messages when
developing/testing a site locally. `ReportingObserver` provides the solution to
this problem. When users experience potential issues in the wild, web developers
can be notified about them.

`ReportingObserver` has only shipped in Chrome 69. It is being considered by
other browsers.
{: .dogfood }

## Background

A while back, I wrote a blog post ("[Observing your web
app](https://ericbidelman.tumblr.com/post/149032341876/observing-your-web-app)")
because I found it fascinating how many APIs there are for monitoring the
"stuff" that happens in a web app. For example, there are APIs that can observe
information about the DOM:
[`ResizeObserver`](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver),
[`IntersectionObserver`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver),
[`MutationObserver`](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver).
[`PerformanceObserver`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver)
captures performance measurements. And methods like
[`window.onerror`](https://developer.mozilla.org/en-US/docs/Web/API/GlobalEventHandlers/onerror)
and
[`window.onunhandledrejection`](https://developer.mozilla.org/en-US/docs/Web/API/WindowEventHandlers/onunhandledrejection)
even let us know when something goes wrong.

However, there are other types of warnings which are not captured by the
existing APIs. When your site uses a deprecated API or runs up against a
[browser intervention][interventions], DevTools is the first to tell you about
them:

<figure>
  <img src="./consolewarnings.png"
       class="screenshot" alt="DevTools console warnings for deprecations and interventions."
       title="DevTools console warnings for deprecations and interventions.">
  <figcaption>Browser-initiated warnings in the DevTools console.</figcaption>
</figure>

One would naturally think `window.onerror` captures these warnings. It does not.
That's because `window.onerror` does not fire for warnings generated directly by
the user agent itself. It fires for runtime errors (JS exceptions and syntax
errors) caused by code execution.

`ReportingObserver` picks up the slack. It provides a programmatic way to be
notified about browser-issued warnings such as [deprecations][deprecations] and
[interventions][interventions]. You can use it as a reporting tool and lose less
sleep wondering if users are hitting unexpected issues on your live site.

{% Aside 'key-term' %}
`ReportingObserver` is part of a larger spec, the [Reporting
API](/web/updates/2018/09/reportingapi), which provides a common way to send
these different reports to a backend. The Reporting API is a generic framework
to specify a set of server endpoints to report issues to.
{% endAside %}

## The API

`ReportingObserver` is not unlike the other "observer" APIs such as
`IntersectionObserver` and `ResizeObserver`. You give it a callback; it gives
you information. The information that the callback receives is a list of issues
that the page caused:

```js
const observer = new ReportingObserver((reports, observer) => {
  for (const report of reports) {
    // → report.type === 'deprecation'
    // → report.url === 'https://reporting-observer-api-demo.glitch.me'
    // → report.body.id === 'XMLHttpRequestSynchronousInNonWorkerOutsideBeforeUnload'
    // → report.body.message === 'Synchronous XMLHttpRequest is deprecated...'
    // → report.body.lineNumber === 11
    // → report.body.columnNumber === 22
    // → report.body.sourceFile === 'https://reporting-observer-api-demo.glitch.me'
    // → report.body.anticipatedRemoval === <JS_DATE_STR> or null
  }
});

observer.observe();
```

### Filtered reports

Reports can be pre-filtered to only observe certain report types. Right now,
there are two report types: `'deprecation'` and `'intervention'`.

```js
const observer = new ReportingObserver((reports, observer) => {
  ...
}, {types: ['deprecation']});
```

### Buffered reports

Use the The `buffered: true` option when you want to see the reports that were
generated before the observer instance was created:

```js
const observer = new ReportingObserver((reports, observer) => {
  ...
}, {types: ['intervention'], buffered: true});
```

This option is great for situations like lazy-loading a library that uses a
`ReportingObserver`. The observer gets added late, but you don't miss out on
anything that happened earlier in the page load.

### Stop observing

Stop observing using the `disconnect()` method:

```js
observer.disconnect();
```

## Examples

### Report browser interventions to an analytics provider

```js
const observer = new ReportingObserver((reports, observer) => {
  for (const report of reports) {
    sendReportToAnalytics(JSON.stringify(report.body));
  }
}, {types: ['intervention'], buffered: true});

observer.observe();
```

### Be notified when APIs are going to be removed

```js
const observer = new ReportingObserver((reports, observer) => {
  for (const report of reports) {
    if (report.type === 'deprecation') {
      sendToBackend(`Using a deprecated API in ${report.body.sourceFile} which will be
                     removed on ${report.body.anticipatedRemoval}. Info: ${report.body.message}`);
    }
  }
});

observer.observe();
```

## Conclusion

`ReportingObserver` gives you an additional way for discovering and monitoring
potential issues in your web app. It's even a useful tool for understanding the
health of your code base (or lack thereof). Send reports to a backend, know
about the real-world issues, update code, profit!

## Future work {: #future }

In the future, my hope is that `ReportingObserver` becomes the de-facto API for
catching all types of issues in JavaScript. Imagine one API to catch everything
that goes wrong in your app:

- [Browser interventions][interventions]
- Deprecations
- [Feature Policy][featurepolicy] violations. See [crbug.com/867471](https://bugs.chromium.org/p/chromium/issues/detail?id=867471).
- JS exceptions and errors (currently serviced by `window.onerror`).
- Unhandled JS promise rejections (currently serviced by `window.onunhandledrejection`)

**Additional resources**:

- [W3c spec][reportingobserver]
- [chromestatus.com entry][chromestatus]

Photo by [Artem
Maltsev](https://unsplash.com/@art_maltsev?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on
[Unsplash](https://unsplash.com/s/photos/report?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText).

[spec]: https://w3c.github.io/reporting
[reportingobserver]: https://w3c.github.io/reporting/#observers
[explainer]: https://github.com/W3C/reporting/blob/master/EXPLAINER.md
[chromestatus]: https://www.chromestatus.com/feature/4691191559880704
[featurepolicy]: /web/updates/2018/06/feature-policy
[interventions]: https://www.chromestatus.com/features#intervention
[deprecations]: https://www.chromestatus.com/features#intervention