# Explainer: the `privacy-screen` HTML attribute

## Table of contents

- [Introduction](#introduction)
- [The problem, in one picture](#the-problem-in-one-picture)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User-facing motivation](#user-facing-motivation)
- [Proposed solution](#proposed-solution)
- [How it works](#how-it-works)
- [Key design decisions](#key-design-decisions)
- [Considered alternatives](#considered-alternatives)
- [Privacy & security self-review](#privacy--security-self-review)
- [Accessibility](#accessibility)
- [Stakeholder feedback / opposition](#stakeholder-feedback--opposition)
- [Open questions](#open-questions)
- [References & acknowledgements](#references--acknowledgements)

## Introduction

Modern devices are starting to ship **dynamic, regional privacy displays** — screens that can, in hardware or at the compositor level, narrow the emission angle of pixels so content is readable head-on but not from the side, and can do so for *part* of the screen rather than all of it. This directly attacks "shoulder surfing," the everyday leak of sensitive on-screen content to whoever is sitting next to you.

The gap: the web has no way to cooperate with this capability. The OS control is global and manual. A page that *knows* which of its content is sensitive — a one-time passcode, a balance, a private thread — can't tell the platform "please make *this* discreet."

We propose a tiny primitive to close that gap: a global boolean HTML attribute, `privacy-screen`. Putting it on an element asks the user agent to request that the OS apply its privacy-display effect over the region that element occupies. It is a **hint** with no guarantee and, crucially, **no readback** — a page can ask, but can never learn whether it worked, which is what keeps the feature from becoming a fingerprinting vector or a source of false assurance.

## The problem, in one picture

> You are on a packed commuter train. Your bank texts a one-time code. To read it and type it in, you expose it — and the balance beneath it — to two strangers reading over each shoulder. You tilt the phone, cup your hand, and hope. The device *has* hardware that could make just that field unreadable from the side, but your banking web app has no way to switch it on for that field.

## Goals

- Let authors **declaratively mark** a region as sensitive so the user agent can request an OS privacy-display effect over it.
- Be **progressively enhancing**: perfect no-op on devices/UAs without the capability; identical page behavior everywhere.
- Add **zero fingerprinting surface** and **zero false-assurance surface** — nothing about the device or the outcome is observable to content.
- Keep the **user and user agent in ultimate control**.
- Be the **smallest testable primitive** that does the job (per TAG "add capabilities carefully").

## Non-goals

- **Not** a guarantee of confidentiality. Off-axis legibility depends on angle, brightness, and environment; vendors caution content can remain partly visible.
- **Not** a way to hide content from the *user*, from assistive technology, from screen capture, or from a consenting head-on viewer. (`hidden`/`inert`/field masking already exist for content-level hiding.)
- **Not** a device-capability query. There is intentionally no "is privacy display available/active?" API.
- **Not** a DRM / anti-screenshot / anti-recording feature.
- **Not** a graduated "strength" control.

## User-facing motivation

Concrete scenarios the primitive is meant to serve:

1. **One-time codes & passwords.** Mark the OTP/password field; the surrounding strangers can't read the code as you type it.
2. **Financial summaries.** Mark the balance/transactions block on a banking or wallet page.
3. **Private messaging.** A web messenger marks the message pane so a thread isn't legible to a seatmate.
4. **Health & identity.** Test results, ID numbers, addresses shown in a public queue.
5. **Notifications-style ephemeral UI** rendered by web apps (e.g. an in-page toast showing a verification code).

These mirror the real-world scenarios device vendors cite for the capability (transit, cafés, waiting rooms, airport lounges).

## Proposed solution

A global boolean attribute. Presence = "request privacy display over this element's region."

```html
<!-- A sensitive field -->
<input autocomplete="one-time-code" inputmode="numeric" privacy-screen>

<!-- A sensitive region -->
<section class="balance" privacy-screen> … </section>
```

It reflects as a boolean IDL property, so it can be toggled from script — but toggling only sets *intent*; it returns no information about the device:

```js
thread.privacyScreen = true;   // request it
thread.privacyScreen = false;  // withdraw the request
```

Cross-origin embeds are gated by Permissions Policy (default `'self'`), delegated explicitly:

```html
<iframe src="https://wallet.example/widget" allow="privacy-screen"></iframe>
```

That's the whole surface. No new interface, event, promise, or media query.

## How it works

1. The attribute defines a **privacy region**: the area the element (and its rendered descendants) occupies in the viewport.
2. On each rendering update, the UA unions the privacy regions of all elements whose request is *active* — meaning the attribute is set, the element is actually rendered, the document is **visible**, the page is a **secure context**, and Permissions Policy allows it.
3. The UA emits a single **privacy screen request** for that combined region to the OS.
4. The OS applies its privacy-display effect if it can. If it can't (no hardware, user turned it off, casting in progress, thermal limits…), it's a **silent no-op**.
5. Nothing in steps 3–4 is ever reported back to the page.

Because reconciliation rides the rendering loop, the region automatically follows scrolling, layout changes, and visibility changes with no author-visible events.

## Key design decisions

### It's a hint, not a command — and the outcome is invisible

This is the load-bearing decision. Two reasons:

- **Fingerprinting.** If a page could detect that privacy display is available or active, that reveals it's a specific class/model of device — a high-entropy bit. So the feature is *write-only*.
- **False assurance.** If a page (or the UA on the page's behalf) could truthfully say "you're protected now," it would encourage users to lower their guard, when the vendor's own documentation says content can still leak at some angles. A feature that can't confirm success can't lie about it.

The cost is real: you can't build an accurate "privacy on" indicator from web content. We think that's the right trade, and we call it out as [an open question](#open-questions) for anyone who disagrees.

### Declarative and semantic

"This content is sensitive" is a property of the *content*, known to the author, and stable across styling. A boolean attribute expresses that intent directly and works without script. (Whether the *mechanism* nonetheless belongs in CSS is discussed below.)

### Tie to visibility and secure contexts

A backgrounded page must not affect what a foreground app shows, so requests require `visibilitychange` = visible. And because this is a privacy-adjacent platform affordance, it's restricted to secure contexts, per TAG guidance.

### Reuse existing machinery

Boolean attribute + reflection + Permissions Policy + Page Visibility. No bespoke lifecycle. Easier to review, spec, and implement interoperably.

### Conformance

The spec allows the author to hint at a sensitive *element*. However, not all combinations of hardware and software may be able to render just one element. In order to conform to the spec, the UA can try each possibility in this list:

* **If there is no hardware support:** the UA **is conforming**.
* **If there is hardware support:**
  - the UA **is conforming** if it can render just the pixels of *the element* with the hardware.
  - Failing this, the UA **is conforming** if it can render just the pixels of the *bounding box* of element with the hardware.
  - Failing this, the UA **is conforming** if it can render *the viewport* with the hardware.
  - Failing this, the UA **is conforming** if it can render *the window* with the hardware.
  - Failing this, the UA **is conforming** if it can render *the screen* with the hardware.
  - Failing this, the UA **is not conforming**.
  
## Considered alternatives

### 1. A CSS property (e.g. `privacy-screen: on`)

**Pro:** off-axis legibility is arguably presentational; CSS gives free integration with media queries, cascade, and `@supports`; it composes with `content-visibility`.
**Con:** `@supports (privacy-screen: on)` would leak capability (fingerprinting) unless deliberately made to always match; the intent is semantic ("sensitive") rather than stylistic; harder for non-CSS authors.
**Status:** Strong contender for the *underlying mechanism*; recorded as an open layering question. The HTML attribute was chosen first for authoring clarity and because it keeps capability-detection (`@supports`) off the table by construction.

### 2. A JavaScript API (`navigator.privacyScreen.request(element)`)

**Pro:** familiar; room for future options.
**Con:** invites a promise/return value that reports success → reintroduces exactly the fingerprinting and false-assurance problems we're avoiding; requires script; more surface to secure. Rejected for v1.

### 3. A `<meta>` tag / whole-page switch

**Con:** the capability's whole point is *regional*; a page-level switch throws that away and is coarser than the OS's own global toggle already is. Rejected.

### 4. A dedicated element (`<private>…</private>`)

**Con:** privacy is a cross-cutting quality of arbitrary existing content, not a new content category; a wrapper element distorts document semantics and layout. A global attribute (like `hidden`, `inert`) is the right shape. Rejected.

### 5. Do nothing / leave it to the OS

**Con:** the OS toggle is global and manual; it can't know *which* part of a web page is the OTP field. Only the document has that knowledge. Doing nothing wastes the regional capability the hardware provides. Rejected.

## Privacy & security self-review

Answers to the TAG *Security & Privacy Questionnaire* (abridged):

- **What information does this expose, and is it necessary?** None. The feature is write-only; no device or outcome state is exposed.
- **Do you expose a new fingerprinting bit?** No — by explicit normative requirement ([spec §9](./privacy-screen-attribute-spec.md#9-non-observability-requirement)).
- **What happens in a private/incognito context?** Identical behavior; nothing is stored.
- **Does it persist state?** No.
- **Is it available to third-party contexts?** Only via Permissions Policy delegation (default `'self'`).
- **What's the abuse potential?** Low: the effect can't hide content from the user or from assistive tech, can't defeat screen capture, and can't be read back. Main risk is *false assurance*, mitigated by making success unobservable.
- **What happens on unsupported devices?** Silent no-op; pages behave identically.

## Accessibility

- No effect on the accessibility tree, focus, or interaction.
- Users for whom the physical effect is uncomfortable (e.g. brightness/contrast sensitivity, or those who rely on viewing at an angle) can disable honoring globally at the UA/OS level; the page can't force it on regardless.
- It is **not** a substitute for content-level hiding (`hidden`, `inert`, masked inputs).

## Stakeholder feedback / opposition

*To be gathered during incubation.* Anticipated discussion points, seeded here for honesty:

- **Browser engines:** likely to probe the fingerprinting story hard — the non-observability requirement is the answer, and should be stress-tested (timing side channels, paint differences).
- **CSS WG / TAG:** likely to ask "why HTML and not CSS?" — see [alternatives](#considered-alternatives) and open questions.
- **Accessibility community:** interest in the low-vision / angle-viewing interaction and the global user override.
- **Privacy advocates:** generally favorable (user-protective, no data flow), provided readback stays off.
- **OS/hardware vendors:** implementation partners; granularity of regional support varies and will shape what UAs can honor.

## Open questions

1. Naming: `privacy-screen` vs `privacyscreen` vs `privacy-display` vs `discreet`.
2. Layer: HTML attribute vs CSS property (or both, with the attribute as sugar).
3. Permissions Policy default: `'self'` vs `'*'`.
4. Any user-mediated (permission-prompted) way to expose "honored," to enable an accurate indicator — accepting the fingerprinting cost?
5. Graduated strength (a "maximum" mode) without creating a capability signal.
6. Behavior under fullscreen, Picture-in-Picture, and multi-screen.

## References & acknowledgements

- Key words per BCP 14 — RFC 2119 & RFC 8174.
- W3C TAG *Web Platform Design Principles*; W3C *Privacy Principles*; TAG *Security & Privacy Self-Review Questionnaire*.
- HTML Standard; Permissions Policy; Page Visibility; Secure Contexts.
- Motivated by shipping and announced regional privacy-display capabilities from consumer display/OS vendors (e.g. hardware "Flex Magic Pixel"–class OLED privacy displays supporting whole-screen, per-app, notification-only, and partial-region privacy).

Thanks to the TAG principles authors and to the vendors whose regional privacy-display work makes a per-element web hint meaningful.
