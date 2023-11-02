# Origin Isolation and Deprecating `document.domain`

**Authors**: [Daniel Vogelheim](https://github.com/otherdaniel/), [Mike West](https://github.com/mikewest/)

## Background

The primary security boundary of the World Wide Web is the
[origin](https://html.spec.whatwg.org/multipage/origin.html#origin). The
[same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
guarantees that one web page cannot access (modify, or extract data
from) another page, unless those pages are hosted on the same origin.
Several pages within an origin can fully cooperate as a single website,
but pages from different origins are isolated and cannot
interfere with each other.

The same origin policy is not just a proven and effective security boundary,
it is also intuitively understood by even novice users.

Unfortunately, the details are substantially more complex. One particular
complexity revolves around the
[Spectre](https://en.wikipedia.org/wiki/Spectre_%28security_vulnerability%29)
family of attacks and the `document.domain` accessor:

Modern browsers [isolate different sites from each other](https://www.chromium.org/Home/chromium-security/site-isolation), by separating execution into
different operating system processes. Pages that need to cooperate need to
be assigned to the same process. Pages that do not cooperate can be assigned
to different processes.

## A Problem and a Solution

The Spectre attacks undermine this model, since they allow near-arbitrary
memory reads *within the same process*. We have a high-level
security boundary based on origins - the same-origin policy - and a low-level
security boundary based on the operating system processes. Spectre exposes a
misalignment between them: The browser wants to enforce the same-origin policy,
but if Spectre can be used to read data from anywhere in the same process,
then this will evade the same-origin policy if
[different origins are assigned to the
same process](https://chromium.googlesource.com/chromium/src/+/master/docs/security/side-channel-threat-model.md#multiple-origins-within-a-siteinstance).
In practical terms, a script on an arbitrary domain could use
Spectre to read confidential data - like login passwords - from another origin
that happens to be assigned to the same process.

In other words, the problem is that the user-visible high-level security
boundaries and the low-level, system security boundaries are misaligned. The
solution is to align them, so that process isolation follows the origin
boundary.

This would appear to be easily done, since process isolation is invisible
to the APIs and therefore browsers are free to change their allocation
strategies at will. Except for one particular "trick":

Setting the `document.domain` accessor allows pages (within a site,
e.g. user1.example.com and user2.example.com) access to each other.
To implement this access, the data from those origins needs to be in the same
process. A example use case that we have observed are pages that host video
content in an iframe, where the iframe is served from a different origin than
the main page. By setting document.domain to their common domain suffix,
these can give each other access and e.g. the page can directly control the
player. To accomodate this usage, browsers will allocate those two pages
to the same process.

[Setting document.domain has been deprecated](https://html.spec.whatwg.org/multipage/origin.html#relaxing-the-same-origin-restriction)
for a long time, but continues to be supported by browsers.
This forces process allocation to be by site,
not by origin, because a page *might* use `document.domain` to get cross-origin
access - within the site - later on. Measurements indicate that [enabling
cross-origin access by setting `document.domain` happens on around 0.5% of page views](https://chromestatus.com/metrics/feature/timeline/popularity/2544)  So 99% of pages do not even use this feature. But because they *might* want
to set `document.domain` later on, browsers have to allocate processes by site.

In other words: > 99% of page views pay for a feature - in terms of reduced
security - that they don't make any use of.

## A Proposal

The [`Origin-Agent-Cluster` http header](https://web.dev/origin-agent-cluster/)
([spec](https://html.spec.whatwg.org/multipage/origin.html#origin-keyed-agent-clusters))
allows a page to request being isolated by origin (instead of site). If set
`true` (`Origin-Agent-Cluster: ?1`), the browser is asked to isolate pages by
origin. If `false`, by site.
(Agent Cluster is spec speak for isolation groups. Since the low-level
isolation is not visible at the API layer, specifications only cursorily
touches the subject.) As a side effect, writing to the  document.domain`
accessor is ignored.

Currently, absence of the `Origin-Agent-Cluster` header defaults to `false`,
meaning that an absent header forces clustering by site, rather than by origin,
and setting `document.domain` continues to work. The proposal here is to change
this default.

In detail:

* The `Origin-Agent-Cluster:` header will, when present, continue to work as
  it currently does. What will change is the default when the header is absent.
* We'll implement a console warning when a page assigns to `document.domain`
  but does not set an `Origin-Agent-Cluster: ?0` header.
* We'll build a feature flag that allows developers to opt-into the new default behavior
  locally in order to track down issues on their own sites.
* Then we wait.
* After developers have had some time to adjust, change the flag's default
  value.
* More waiting. Then remove the flag. Now the transition is
  complete, and the only way to relax the same-origin policy through
  document.domain is to send an `Origin-Agent-Cluster: ?0` header.
* (Chrome-specific:) We expect to have an admin-setting for this flag, which
  would likely remain long-term.

## FAQ

### Who needs to set `Origin-Agent-Cluster` header?

Any page that wishes to set `document.domain` will need to opt-into the ability to do so by sending `Origin-Agent-Cluster: ?0`. This includes both top-level documents that wish to synchronously script each other, as well as frames that wish to opt into the same relaxed security posture.

### What should developers use instead?

`window.postMessage()` provides an explicit communication channel for cross-origin collaboration.

This isn't a perfect match for some use cases which require user activation to flow from one frame to another (consider a [video player in a cross-origin `<iframe>`](https://twitter.com/JibberJim/status/1318134009252237312)), but is the right general purpose mechanism to point developers towards. Delegation of capability that user activation enables might be possible in the future via a mechanism like the one described in [mustaqahmed/capability-delegation](https://github.com/mustaqahmed/capability-delegation).

### Do we know anything useful about the ~0.4% usage noted above?

In the 2020-12-01 HTTP Archive corpus, we see 7038 unique pages (of 7,849,064: 0.09%) whose behavior was influenced by `document.domain`.

<details>
   <summary>HTTP Archive Data</summary>

Raw data produced by the following query is available in CSV format at https://github.com/mikewest/deprecating-document-domain/blob/main/2020-12-document-domain-usage.csv.

```sql
SELECT
  url, NET.REG_DOMAIN(url) as host
FROM
  (
    SELECT * FROM httparchive.pages.2020_12_01_desktop
    UNION ALL
    SELECT * FROM httparchive.pages.2020_12_01_mobile
  )
WHERE
  # DocumentDomainEnabledCrossOriginAccess
  JSON_EXTRACT(payload, '$._blinkFeatureFirstUsed.Features.2544') IS NOT NULL
  # DocumentDomainBlockedCrossOriginAccess
  OR JSON_EXTRACT(payload, '$._blinkFeatureFirstUsed.Features.2543') IS NOT NULL
GROUP BY
  url
ORDER BY
  host ASC
```

</details>

Skimming through the data, a few examples seem worth poking at:

* Alibaba runs storefronts at `*.alibaba.com` domains (i.e. `https://baominhjsc.trustpass.alibaba.com/`) that rely on `document.domain` to communicate with a messenger widget from `onetalk.alibaba.com`. 836 such pages show up in this run of the corpus. `iven.co.kr` (e.g. `http://pad.inven.co.kr/`) has a similar setup with ~123 subdomains, as do `58.com` (e.g. `https://bd.58.com/`) with 90 subdomains, `sutochno.ru` (e.g. `https://nv.sutochno.ru/`) with 92 subdomains, and a few others (`fang.com` => 77, `diary.ru` => 77, `idnes.cz` => 71, and so on).

* `qq.com` has ~110 subdomains that rely on `document.domain` to communicate login status with frames like `apps.game.qq.com`.

* `58.com` has 90 subdomains using a messaging widget

* `bbc.com` loads a media player from `emp.bbc.com`. It doesn't look essential; the page continues to work after setting `document.domain` to something else in the console.

    * Based on some [Twitter conversation](https://twitter.com/mikewest/status/1318100840247427078) it might be possible to remove this dependency.

* `www.naver.com` depends on it more directly, from a `siape.veta.naver.com` frame. It appears that it's doing viewport checks which `IntersectionOberver` might better address?

* `www.sina.com.cn` has a lazy-load script that seems to do viewport checks; `IntersectionObserver` or `loading=lazy` might be better fits.

### Does disabling `document.domain` suffice to enable origin-level process isolation?

No. As noted above, `document.domain` is one (large!) blocker among several. This is one step along the road to enabling isolation by default. The [`origin-isolation` explainer](https://github.com/WICG/origin-isolation#how-it-works) points to additional steps that will be essential to splitting sites into distinct agent clusters. Disabling `document.domain` is necessary, but not sufficient.

### What about the `document-domain` Feature Policy?

Web developers are able to opt-out of `document.domain` usage today via the [`document-domain` Feature Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Feature-Policy/document-domain). Some recent changes to Permission/Feature Policy's inheritance rules make this difficult to use as a deprecation mechanism, however. Had Document Policy existed when we introduced the feature, it would have been a better option. As-is, we should likely remove that policy from Feature Policy and Permissions Policy in favor of the document-based mechanism we now have available.
