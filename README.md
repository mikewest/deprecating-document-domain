# Deprecating `document.domain`

## A Problem

Given that computers are broken, it's reasonable to assume a threat model which posits that it is [impossible to create a security boundary inside a single address space](https://chromium.googlesource.com/chromium/src/+/master/docs/security/side-channel-threat-model.md). It's incumbent upon us, then, to align browsers' process model to the web's security boundaries. [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation) is a big step in that direction, allowing origins to be partitioned by eTLD+1, cleanly separating `attacker.example` from `secrets.example`. Site-level isolation is, however, insufficient as [same-site-but-cross-origin attacks](https://chromium.googlesource.com/chromium/src/+/master/docs/security/side-channel-threat-model.md#multiple-origins-within-a-siteinstance) remain a real threat. Ideally, we'd map the process boundary to an origin, not a site.

There are several challenges to shipping origin-level isolation, but the clearest blocker is the impact that such isolation would have on the ability to relax the same-origin policy through `document.domain`. Using this mechanism, same-site-but-cross-origin documents can decide they should be able to synchronously script each other at runtime, which makes it impossible to commit them into distinct processes _a priori_. In addition to preventing broad adoption of origin-level process isolation, this feature is bad in itself, as it significantly complicates both the implementation of the web's security boundary, and the expectations we can reasonably set for web developers.

Despite [admonitions against its usage in the HTML spec](https://html.spec.whatwg.org/multipage/origin.html#relaxing-the-same-origin-restriction), the setter is unfortunately quite pervasive, affecting at least [~0.4% of page views in Chrome](https://chromestatus.com/metrics/feature/timeline/popularity/2544) (down from ~6% thanks to [Facebook's heroic efforts](https://twitter.com/mikewest/status/1136861248186998784)).

## A Proposal

Accepting the post-Spectre threat model means that we must begin work in earnest to unblock origin-level process isolation by removing the `document.domain` setter from the platform, sooner rather than later. Given its broad usage, an iterative approach to removal seems reasonable.

We can begin with simple subsetting: the setter's effect on a origin's [domain](https://html.spec.whatwg.org/#concept-origin-domain) should be a no-op in the presence of `Cross-Origin-Opener-Policy` headers, as it already is for `Origin-Isolation` and `crossOriginIsolated` documents.

Following that, we can create a ratcheting deprecation mechanism by introducing a `document-domain` Document Policy configuration point, with a default value of `false` (after an appropriate period of developer-facing messaging in places like browsers developer console, Lighthouse, etc). This policy would similarly disable `document.domain` by default, while still providing developers the ability to opt-into insecurity by sending an explicit `Document-Policy: document-domain` header along with their response. This explicit opt-in will allow us to track usage via HTTP Archive and browser telemetry, and to make appropriate decisions about a document's process when committing a navigation.

As we reduce usage over time, we can strengthen this initial shift by requiring additional opt-in. Enterprise policies and reverse origin trials come to mind as subsequent milestones.

## FAQ

### Who needs to opt-into the `document-domain` Document Policy?

Any page that wishes to set `document.domain` will need to opt-into the ability to do so by sending `Document-Policy: document-domain`. This includes both top-level documents that wish to synchronously script each other, as well as frames that wish to opt-into the same relaxed security posture.

### What should developers use instead?

`window.postMessage()` provides an explicit communication channel for cross-origin collaboration.

This isn't a perfect match for some use cases which require user activation to flow from one frame to another (consider a [video player in a cross-origin `<iframe>`](https://twitter.com/JibberJim/status/1318134009252237312)), but is the right general purpose mechainsm to point developers towards. Delegation of capability that user activation enables might be possible in the future via a mechanism like the one described in [mustaqahmed/capability-delegation](https://github.com/mustaqahmed/capability-delegation).

### Do we know anything useful about the ~0.4% usage noted above?

Trawling through HTTP Archive, we see 3,986 (of 5,438,156: 0.07%) pages in the 2020-09-01 desktop corpus, and 2,650 (of 6,845,384: 0.04%) in the mobile corpus. I've compiled these [into a spreadsheet](https://docs.google.com/spreadsheets/d/1jERqy1Up1bdHH5SZhy7e0qxaZY7IFBkEpgvqYyGtiMw/edit?usp=sharing) for easier parsing.

Skimming through the data, a few examples seem worth poking at:

* Alibaba runs storefronts at ~950 `*.alibaba.com` domains (i.e. `https://baominhjsc.trustpass.alibaba.com/`) that rely on `document.domain` to communicate with a messenger widget from `onetalk.alibaba.com`. `58.com` and `fang.com` seem to have similar arrangements.

* `qq.com` has ~120 subdomains that rely on `document.domain` to communicate login status with frames like `apps.game.qq.com`.

* `bbc.com` loads a media player from `emp.bbc.com`. It doesn't look essential; the page continues to work after setting `document.domain` to something else in the console.

    * Based on some [Twitter conversation](https://twitter.com/mikewest/status/1318100840247427078) it might be possible to remove this dependency.

* `www.naver.com` depends on it more directly, from a `siape.veta.naver.com` frame. It appears that it's doing viewport checks which `IntersectionOberver` might better address?

* `www.sina.com.cn` has a lazy-load script that seems to do viewport checks; `IntersectionObserver` or `loading=lazy` might be better fits.

### Does disabling `document.domain` suffice to enable origin-level process isolation?

No. As noted above, `document.domain` is one (large!) blocker among several. This is one step along the road to enabling isolation by default. The [`origin-isolation` explainer](https://github.com/WICG/origin-isolation#how-it-works) points to additional steps that will be essential to splitting sites into distinct agent clusters. Disabling `document.domain` is necessary, but not sufficient.

### What about the `document-domain` Feature Policy?

Web developers are able to opt-out of `document.domain` usage today via the [`document-domain` Feature Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Feature-Policy/document-domain). Some recent changes to Permission/Feature Policy's inheritance rules make this difficult to use as a deprecation mechanism, however. Had Document Policy existed when we introduced the feature, it would have been a better option. As-is, we should likely remove that policy from Feature Policy and Permissions Policy in favor of the document-based mechanism we now have available.
