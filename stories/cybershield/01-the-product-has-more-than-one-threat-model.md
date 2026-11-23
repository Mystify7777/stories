# Chapter 1: The Product Has More Than One Threat Model

CyberShield brings several kinds of trust work into one product. A user may report an incident, a moderator may review it, someone may ask whether a message looks like a scam, or a user may submit a URL for passive analysis. These flows share a security theme, but they do not share identical risks.

The repository describes a product with authentication, role-based administration, report lifecycle management, community content, an AI-assisted text detector, and TrustScan. The result is a broad application rather than a single-purpose scanner. That breadth is useful, but it creates an engineering obligation: each capability needs its own trust boundary, data rules, and honest description of what it can establish.

## Two different questions

The AI text detector asks whether the text contains patterns associated with scam messages. TrustScan asks what can be inferred from passive signals about a URL: certificate and TLS properties, DNS/domain information, HTTP security headers, and public reputation feeds.

Neither question is equivalent to “is this definitely safe?” A keyword detector can miss a carefully written scam or flag harmless text. A domain can have valid TLS and still host fraud. A reputation service can be unavailable or have incomplete coverage. CyberShield's documentation increasingly frames TrustScan around explainability and confidence rather than a naked score.

## The product boundary

The TrustScan policy explicitly permits passive checks and rules out brute force, exploit attempts, authenticated penetration testing, credential stuffing, and denial-of-service-style traffic. It also requires target validation, rejection of prohibited targets, per-user and per-IP throttling, and clear user-facing language about limitations.

That is an important product decision. TrustScan is meant to inform a defensive decision, not to prove exploitability or perform unauthorized access. Expanding the checks is supposed to require security review, legal review, a policy update, and a release note.

## Evidence and interpretation

The source documents establish intended behavior. The current implementation and tests must still be used to verify each claim. For example, the README lists report sharing and PDF export as capabilities, while the code shows how the public report route and client-side PDF export are implemented. Those details matter more than a feature bullet when the goal is to explain engineering work.

**Sources:** [Product specification](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/product/TRUSTSCAN_PRODUCT_SPEC.md), [TrustScan policy](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/security/TRUSTSCAN_POLICY.md), [API reference](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/engineering/API_REFERENCE.md), [public README](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/README.md).
