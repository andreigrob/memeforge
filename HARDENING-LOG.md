# Part A
1.
HTTP/1.1 200 OK
Date: Fri, 13 Nov 2026 21:16:37 GMT
Connection: close
X-Powered-By: PHP/8.6.0RC2
Set-Cookie: PHPSESSID=0467f8c4a88c1b83706b535c093063ef; path=/; HttpOnly; SameSite=Lax
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-type: text/html; charset=UTF-8

2.
The security vulnerability presented by the cookie implementation is that SameSite=Lax is set, which allows the cookie to be sent to third-party websites.

3.
HTTP/1.1 200 OK
Date: Fri, 13 Nov 2026 21:20:23 GMT
Connection: close
X-Powered-By: PHP/8.6.0RC2
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-type: text/html; charset=UTF-8

The response header is the same as the previous one, except for Set-Cookie, which is missing in this response.

# Part B

The real memeforge page is loaded as a frame in the attacker's page. The user then clicks on the button on the memeforge page when they click on the button on the attacker's page. 

trick revealed: one.png
before: two.png


# Content-Security-Policy

| # | Blocked resource | Page(s) | Directive | Fix |
|---|---|---|---|---|
| 1 | `https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css` | All pages | `style-src` (from `default-src`) | **Self-hosted** at `public/assets/vendor/bootstrap/bootstrap.min.css`. The download's sha384 matches Bootstrap's published SRI (`sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH`). |
| 2 | `https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js` | All pages | `script-src` (from `default-src`) | **Self-hosted** at `public/assets/vendor/bootstrap/bootstrap.bundle.min.js`. The download's sha384 matches the published SRI (`sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz`). |
| 3 | Inline `<style>` block (`.navbar-brand .brand-mark { color: #ffc93c; }`) | All pages (`render_header`) | `style-src` (`style-src-elem`) | Moved the rule into `public/assets/app.css`. |
| 4 | Inline `style="top: 1rem; z-index: 1;"` on the preview column | New Meme, Edit | `style-src` (`style-src-attr`) | Replaced with `class="sticky-top preview-sticky"`, and added `.sticky-top.preview-sticky { top: 1rem; z-index: 1; }` to `app.css`. |
| 5 | Inline `<script>` (live caption preview) | New Meme, Edit (`render_preview_script`) | `script-src` (`script-src-elem`) | Moved verbatim to `public/assets/preview.js`, loaded with `<script src="assets/preview.js">`. |
| 6 | `data:image/svg+xml,…` images defined inside Bootstrap's stylesheet: the `.navbar-toggler-icon` hamburger (only shown at phone width), the `.form-select` arrow, and the `.form-check-input` radio dots | All pages at narrow width (navbar); New Meme, Edit (select and radios) | `img-src` (from `default-src`) | Added `img-src 'self' data:`. This is the only place `data:` is allowed. A data: image cannot run script. |

**Why self-host Bootstrap instead of allowing cdn.jsdelivr.net:**  Self-hosting keeps the policy at `'self'` and removes the dependency on a third party (outages, tracking, a compromised CDN).


# Questions

1. CSP defends against different attacks e.g. clickjacking. Also, CSP is a second layer: even when an attacker gets markup into the page, the browser refuses to run inline scripts or load scripts from other origins. A concrete case: a future developer adds a new field to the user page and prints it without e(). A stored `<script>fetch</script>` would be blocked by `script-src 'self'`.

2. An anti-CSRF token stops a different site from forging a request, because the attacker cannot read the secret token and so cannot build a valid form. In clickjacking the attacker does not forge anything: they load the real MemeForge page in an iframe, and the user's own click lands on the real button in the real form. That form was rendered by MemeForge in the user's session, so it already contains the correct token, and the request is legitimate from the server's point of view. Forbidding framing works because the browser refuses to render MemeForge inside another site's frame, so there is no hidden page for the user to click on.

3. Allowing `https://cdn.jsdelivr.net` allows everything on that host. An attacker could publish their own package and load it. The policy would allow it, because it trusts the host. By self-hosting I took on the job of keeping Bootstrap up to date and patching it myself when a vulnerability is announced, and of serving the files reliably. An `integrity="sha384-…"` attribute makes the browser refuse the file if its contents don't match the hash, so a different CDN file cannot run. It protects only the tags we write, though, not tags an attacker injects.

4. Both protections depend on HTTPS, and MemeForge was served over plain `http://` on localhost. Browsers ignore `Strict-Transport-Security` when it arrives over an insecure connection, so the header we send has no effect yet. A session cookie marked `Secure` is only sent over HTTPS, so turning it on over HTTP would make the browser drop the session cookie and break login, which is why it is left commented out. For both to take effect, MemeForge must be served only over HTTPS with a valid certificate, with plain HTTP redirecting to HTTPS.
