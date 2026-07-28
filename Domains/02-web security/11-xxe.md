# XML External Entity (XXE) Injection

## Why this comes right after LFI/RFI
Same underlying pattern as file inclusion — the server reads or fetches something based on attacker-controlled input — but this time the trigger is a feature of the XML format itself, not a file path parameter. Any endpoint that accepts XML input (SOAP APIs, file uploads that parse XML like `.docx`/`.xlsx`/SVG files, or direct XML submissions) is a potential target.

---

## What it is (in plain terms)
XML has a built-in feature called **external entities** — a way to define a placeholder in the document that gets replaced with the contents of a file or URL when the document is parsed. If an application parses user-supplied XML without disabling this feature, an attacker can define an external entity pointing at a local file or internal URL, and the parser will dutifully fetch it and insert its contents into the processed output — reading local files or triggering SSRF, entirely through what looks like a normal data upload.

## Why it exists — the real-life cause

```python
# VULNERABLE — default XML parser settings allow external entities
from lxml import etree

def parse_xml(xml_data):
    tree = etree.fromstring(xml_data)  # external entity resolution enabled by default in many parsers
    return tree
```
An attacker submits XML like this instead of normal data:
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<data>&xxe;</data>
```
If the parser resolves external entities by default (many do, unless explicitly configured not to), `&xxe;` gets replaced with the actual contents of `/etc/passwd`, and that content ends up embedded in whatever the application does with the parsed data next — often reflected back in an error message or the processed output.

```python
# SECURE — explicitly disables external entity resolution
from lxml import etree

def parse_xml(xml_data):
    parser = etree.XMLParser(resolve_entities=False, no_network=True, dtd_validation=False)
    tree = etree.fromstring(xml_data, parser=parser)
    return tree
```
Here, external entity resolution is explicitly turned off at the parser level — even if an attacker submits the exact same malicious XML, the parser simply refuses to fetch or embed the referenced file/URL at all.

## How an attacker actually does it, step by step
1. Find any feature that accepts XML — a direct API endpoint, or indirectly through file formats that are actually XML underneath (`.docx`, `.xlsx`, `.svg` files are all ZIP archives containing XML internally).
2. Submit a payload defining an external entity pointing at a sensitive local file (`file:///etc/passwd`) or an internal URL (same targets as SSRF — `http://169.254.169.254/`, internal admin panels).
3. Check whether the referenced content appears anywhere in the response — directly in the output, or indirectly in an error message that echoes back part of the parsed data.
4. If successful, this becomes a direct route to local file disclosure (same impact as LFI) or SSRF (same impact as file `09`) — XXE is really a delivery mechanism for those same outcomes, just through XML parsing instead of a URL/path parameter.

## Technical Impact
- Local file disclosure — same outcome and severity as LFI (file `10`)
- SSRF — same outcome and severity as file `09`, reachable through a completely different-looking feature (an XML upload, not an obvious "fetch URL" field)
- Denial of Service via the "Billion Laughs" attack — nested entity definitions that expand exponentially, exhausting server memory with a tiny XML payload

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Since XXE ultimately leads to the same outcomes as LFI or SSRF — file disclosure or internal network access — the financial exposure mirrors those: potential credential theft leading into the database or cloud account |
| **Regulatory / compliance** | XXE is frequently under-tested because it hides inside file formats (`.docx`, `.svg`) that don't look like "XML input" on the surface — its presence often signals a gap in the organization's broader input-validation testing coverage, which auditors note as a process gap, not just a single bug |
| **Reputational damage** | Same tier as the LFI/SSRF outcomes it enables — file/credential disclosure leading to broader compromise |
| **Legal liability** | Same reasoning as LFI/SSRF — a preventable parser misconfiguration leading to data exposure is a straightforward negligence argument |
| **Operational cost** | Requires the same investigation depth as LFI/SSRF once confirmed, plus an audit of every feature that touches XML-based file formats — often a wider surface than initially assumed, since `.docx`/`.xlsx`/`.svg` uploads are easy to overlook as "XML input" |

**One-line interview answer:** *"Technically, XXE happens when an XML parser is left with its default settings, which allow it to resolve external entities — letting an attacker read local files or trigger SSRF through what looks like a normal document upload. For a bank, the business impact is the same as LFI or SSRF individually, but the risk is often underestimated because it hides inside ordinary-looking file uploads like Word or Excel documents, not an obviously 'raw XML' field."*

## Mitigation — layered, not just one fix

1. **Disable external entity resolution at the parser level (the real fix)** — as shown above, this should be the default configuration for any XML parser used, not an opt-in setting applied only when a developer remembers to.
2. **Use less-capable data formats where possible** — if XML isn't strictly required, JSON has no equivalent entity-expansion feature and sidesteps this entire vulnerability class by design.
3. **Validate and sanitize file uploads that are secretly XML-based** — `.docx`, `.xlsx`, `.svg`, and similar formats should go through the same hardened parser configuration, since they're easy to overlook as "just a document."
4. **Least-privilege for the parsing process** — limit what the application process can actually read on the filesystem, reducing impact even if a bypass is found.

## Explaining it to a developer
*"Our XML parser is using its default settings, which means it will automatically fetch and embed the contents of any external file or URL an attacker defines inside the XML itself — including things like `/etc/passwd`, or internal-only URLs. This isn't limited to obviously 'XML' features either — Word and Excel files are XML internally, so any upload feature accepting those formats is potentially affected too. The fix is one config change: explicitly disable external entity resolution in the parser, which we should honestly be doing everywhere we parse XML, not just here."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Disable external entity resolution | File disclosure and SSRF via XML parsing |
| Prefer JSON over XML where possible | Removes the vulnerability class entirely by design |
| Harden parsers for `.docx`/`.xlsx`/`.svg` uploads | Closes the "hidden XML" attack surface |
| Least-privilege for the parsing process | Limits impact even if a bypass is found |
