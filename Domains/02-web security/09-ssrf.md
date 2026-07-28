# Server-Side Request Forgery (SSRF)

## Why this comes right after File Upload
File upload features sometimes include an "upload from URL" option — the server fetches a file from a link the user provides instead of the user uploading it directly. That single feature is one of the most common places SSRF shows up, which is why it follows naturally here. More broadly, SSRF applies to *any* feature where the server makes an outbound request based on user input — not just uploads.

---

## What it is (in plain terms)
SSRF happens when an attacker can control (fully or partially) a URL that the *server itself* fetches — meaning the request comes from the server's own network position, not the attacker's browser. This matters enormously because the server usually sits inside a trusted internal network the attacker could never reach directly.

## Why it exists — the real-life cause

```python
# VULNERABLE — fetches whatever URL the user provides, no restrictions
@app.route('/fetch-image')
def fetch_image():
    url = request.args.get('url')
    response = requests.get(url)  # server fetches ANY url, no checks
    return response.content
```
This looks like a harmless "load an image from a URL" feature. But since the server itself makes the request, an attacker can supply an internal address instead of a real image URL — `url=http://169.254.169.254/latest/meta-data/` (a well-known cloud metadata endpoint) or `url=http://localhost:8080/admin` (an internal admin panel not meant to be reachable from the internet) — and the server will fetch it, then return the response to the attacker, effectively using the server as a proxy into its own internal network.

```python
# SECURE — validates and restricts what can be fetched
from urllib.parse import urlparse
import ipaddress

ALLOWED_DOMAINS = ['trusted-image-host.com']

@app.route('/fetch-image')
def fetch_image():
    url = request.args.get('url')
    parsed = urlparse(url)
    if parsed.hostname not in ALLOWED_DOMAINS:
        abort(400)
    ip = socket.gethostbyname(parsed.hostname)
    if ipaddress.ip_address(ip).is_private:  # blocks internal/private IP ranges
        abort(400)
    response = requests.get(url)
    return response.content
```
Here, the destination is checked against an explicit allow-list of trusted domains, and the resolved IP is checked to make sure it doesn't point at an internal/private address — even if the domain itself looked legitimate.

## How an attacker actually does it, step by step
1. Find any feature where the server fetches a URL on the user's behalf — "import from URL," webhook configuration, PDF generation from a URL, image proxy/thumbnail features.
2. Try supplying an internal address instead of an external one: `http://localhost/`, `http://127.0.0.1/`, `http://169.254.169.254/` (cloud metadata service — AWS, Azure, GCP all use addresses in this range).
3. If the response reflects back internal content (an admin panel, internal API response, or cloud credentials from the metadata endpoint), SSRF is confirmed.
4. Escalate — cloud metadata endpoints in particular often return temporary cloud credentials directly, which can then be used to access the cloud account's actual resources (S3 buckets, databases, etc.) well beyond the original web application.

## Technical Impact
- Access to internal-only services and admin panels not meant to be reachable from the internet
- **Cloud credential theft** via metadata endpoints — one of the most severe and common real-world SSRF outcomes, since it can lead directly into the cloud account itself (this is the same underlying risk area as `Finding 3` and `Finding 4` in the cloud-ai-security-assessments repo, from a completely different angle)
- Port scanning the internal network by observing response time/error differences across different internal addresses
- Potential pivot point into internal systems the attacker could never reach directly from the outside

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | If SSRF leads to stolen cloud credentials, the actual financial exposure isn't limited to the web app at all — those credentials could grant access to databases, storage buckets, or other cloud infrastructure, making the blast radius far larger than the original vulnerability suggests |
| **Regulatory / compliance** | SSRF leading to internal network access or cloud credential exposure is treated as a critical finding — it demonstrates the perimeter between "public web app" and "internal trusted network" has failed, which auditors view as a fundamental architecture-level failure, not a simple bug |
| **Reputational damage** | Since SSRF is often a stepping stone to a larger breach (rather than the final impact itself), the eventual damage disclosed publicly is usually whatever the attacker reached *through* the SSRF — which can look far worse than "a web app had a bug" |
| **Legal liability** | Similar to file upload RCE — a compromise that spreads beyond the original application into other infrastructure raises the negligence bar significantly in any legal claim |
| **Operational cost** | Response requires determining exactly what the compromised server could reach internally, rotating any credentials the attacker may have obtained via metadata endpoints, and auditing every downstream system those credentials had access to |

**One-line interview answer:** *"Technically, SSRF lets an attacker make the server itself send requests to internal addresses it wouldn't normally expose to the internet — including cloud metadata endpoints that can return real cloud credentials. For a bank, the business impact is that this isn't limited to the web app at all — it can become a pivot point into internal systems or the cloud account itself, so the actual scope of damage often extends well past the original vulnerable feature."*

## Mitigation — layered, not just one fix

1. **Allow-list trusted destinations (the real fix where feasible)** — only permit requests to a known, explicit set of domains, rather than trying to block a list of "bad" addresses (blocklists are easy to bypass via redirects, DNS tricks, or alternate IP representations).
2. **Block requests to private/internal IP ranges** — explicitly reject `127.0.0.1`, `169.254.169.254`, `10.x.x.x`, `172.16.x.x-172.31.x.x`, `192.168.x.x`, and similar reserved ranges, checking the *resolved* IP, not just the hostname string (since DNS can be manipulated to resolve a normal-looking domain to an internal IP).
3. **Disable unnecessary URL schemes** — restrict to `http`/`https` only; block `file://`, `gopher://`, and other schemes that can be abused for further exploitation.
4. **Use a dedicated, isolated network segment for the fetching service** — even if the check above is bypassed, network-level segmentation limits what the server can actually reach at all.
5. **Never expose sensitive cloud metadata endpoints without IMDSv2/equivalent protections** — modern cloud providers offer hardened metadata service versions specifically designed to resist SSRF-based credential theft; enabling these is a direct mitigation at the infrastructure level.

## Explaining it to a developer
*"This 'fetch a URL' feature makes the request from our own server — which means whatever URL is passed in gets fetched using our server's network access, not the user's. If someone points it at an internal address instead of a real image, our server will fetch that internal resource and hand the response back to them, effectively letting an outside attacker reach things only our internal network should be able to reach. The fix is to only allow fetching from a specific, known list of trusted external domains, and explicitly block anything that resolves to an internal or private IP address."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Domain allow-list | Fetching arbitrary/unexpected destinations at all |
| Block private/internal IP ranges | Reaching internal services, admin panels, metadata endpoints |
| Restrict URL schemes | `file://` and other non-http(s) abuse |
| Network segmentation | Limits reach even if the above checks are bypassed |
| Hardened metadata service (IMDSv2) | Cloud credential theft via metadata endpoint, even if SSRF occurs |
