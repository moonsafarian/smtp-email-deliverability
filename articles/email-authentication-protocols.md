---
title: "The Auth Stack: SPF, DKIM, DMARC, and BIMI for Email Deliverability"
date: 2025-11-18
tags:
  - email
  - deliverability
  - SPF
  - DKIM
  - DMARC
  - BIMI
  - DNS
  - authentication
excerpt: "A deep dive for developers into the DNS records and cryptographic signatures that prove email identity and prevent spoofing."
---

# The Auth Stack: SPF, DKIM, DMARC, and BIMI for Email Deliverability

Email authentication is no longer optional; it's a fundamental requirement for achieving and maintaining good email deliverability. For developers, understanding and correctly implementing Sender Policy Framework (SPF), DomainKeys Identified Mail (DKIM), Domain-based Message Authentication, Reporting, and Conformance (DMARC), and Brand Indicators for Message Identification (BIMI) is crucial. These protocols provide the necessary proof of identity and origin that Mailbox Providers (MBPs) need to trust your emails.

This article provides a technical deep dive into these authentication standards, covering their syntax, cryptographic mechanisms, and policy enforcement strategies.

## DNS Records: The Foundation of Email Authentication

All these authentication technologies rely on information stored within your domain's DNS records, specifically as TXT records. When you register a domain, a DNS table is provided where you can add various "records" or "fields." These records convey additional information about your domain's email handling.

While there's a traditional record (`A` record) that associates your domain with an IP address for web traffic, SPF, DKIM, DMARC, and BIMI require specific TXT records. These records, though not strictly mandatory by protocol, are now **strongly recommended** by MBPs. Their absence can lead to emails being marked as suspicious or sent to spam folders by default.

It's critical that these records are created by the domain owner. You can configure them yourself if you have access to your DNS provider's dashboard, or, as often recommended, work with your DNS provider to publish the records you've prepared.

### The Pitfall of Shared Hosting (Revisited)

Just as with sending IP reputation, shared hosting environments are detrimental to DNS-based email authentication. Even if you meticulously configure your SPF, DKIM, DMARC, and BIMI records, the presence of less diligent co-tenants on a shared IP can negatively impact your email's reputation. MBPs often factor the shared IP's reputation into their scoring, regardless of individual domain configurations.

## SPF (Sender Policy Framework): Authorizing Your Senders

**SPF** allows you to specify which IP addresses and mail servers are authorized to send email on behalf of your domain. By publishing an SPF record, you help recipient mail servers identify legitimate senders and reject unauthorized ones, thereby protecting your domain from phishing and spoofing.

MBPs universally expect an SPF record to be present. Its absence will almost certainly result in your emails being penalized.

### Technical Implementation:

1.  **Identify Sending IPs:** Collect all IP addresses (IPv4 and IPv6) used by any server sending email for your domain. This includes your own servers and any third-party services (e.g., smart hosts, marketing platforms).
2.  **Construct the SPF Record:** Your SPF record is a single TXT string, maximum 255 characters, and should contain a maximum of 10 `include` mechanisms to avoid a "permerror".

    *   Start with `v=spf1`: Specifies the SPF version.
    *   List authorized IPs: `ip4:xxx.xxx.xxx.xxx` or `ip6:aaaa:bbbb::` (space-separated).
    *   Include third-party senders: `include:thirdpartysender.com`. You'll need to consult the third-party provider's documentation for their specific `include` domain.
    *   Define a policy for unauthorized senders (at the end of the record):
        *   `-all` (Hard Fail): Tells MBPs to reject emails from unauthorized senders. This is the strongest policy and generally recommended once confident in your SPF setup.
        *   `~all` (Soft Fail): Tells MBPs to accept emails from unauthorized senders but mark them as suspicious. Useful for initial deployment.
        *   `?all` (Neutral): MBPs treat unauthorized senders as neither allowed nor disallowed. Rarely recommended for production.

    **Example:**
    ```
    v=spf1 ip4:192.0.2.1 include:spf.mailgun.org include:sendgrid.net ~all
    ```
3.  **Publish the SPF Record:** Add this TXT record to your domain's DNS settings. The record name is typically your domain name itself or `@`.

## DKIM (DomainKeys Identified Mail): Cryptographic Signature for Email Integrity

**DKIM** provides a cryptographic signature for each outgoing email, acting as a "digital signature" from your domain. This ensures that the email's content (or specified parts thereof) hasn't been tampered with during transit and verifies the sender's identity. DKIM uses a pair of keys: a private key to sign the message and a public key for verification.

### Technical Implementation:

1.  **Generate DKIM Keys (if self-hosting):** If you manage your own mail server, you'll need to install a DKIM software package (e.g., OpenDKIM) and use its tools to generate a public/private key pair.
    *   **Selectors:** During generation, you'll choose a "selector" (e.g., `s1`, `marketing`). This is a unique identifier that tells recipient servers where to find the public key specific to a sending stream. It's good practice to use meaningful selectors (e.g., `news` for newsletters, `trans` for transactional).
2.  **Publish the Public DKIM Key:** The public key (provided by your DKIM software or smart host) needs to be added as a TXT record in your DNS.
    *   **Record Name:** `[selector]._domainkey.yourdomain.com` (e.g., `s1._domainkey.example.com`).
    *   **Record Value:** This is a specially formatted string from your DKIM provider, typically starting with `v=DKIM1; k=rsa; p=...`.

    **Example (partial):**
    ```
    s1._domainkey.example.com IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3D...IDAQAB"
    ```
3.  **Configure Mail Server (if self-hosting):** Your outbound mail server needs to be configured with the private key to sign outgoing messages. The specific configuration depends on your MTA.
4.  **Multiple Senders:** If you use multiple external services or MTA instances, each often requires its own DKIM key pair and selector. You must repeat this process for every service that sends on behalf of your domain.

**Note:** DKIM signing occurs at the SMTP server level, **not** the email client. When using a smart host, they handle key generation and signing on their infrastructure; you only need to publish the public key they provide.

## DMARC (Domain-based Message Authentication, Reporting, and Conformance): Policy and Reporting

**DMARC** builds upon SPF and DKIM by allowing domain owners to tell recipient mail servers what to do with messages that fail SPF or DKIM authentication (or both). It also provides a reporting mechanism to receive feedback on email authentication results. DMARC standardizes how MBPs should handle emails that don't pass authentication, defining policies such as `none`, `quarantine`, or `reject`.

DMARC is a powerful tool for security and deliverability, but misconfiguration can be highly detrimental.

### Technical Implementation:

1.  **Prerequisites: SPF and DKIM:** SPF and DKIM *must* be correctly implemented and stable before deploying DMARC.
2.  **Construct the DMARC Record:** A DMARC record is a TXT string with various `tag=value` pairs, separated by semicolons.

    *   `v=DMARC1`: The DMARC version (mandatory).
    *   `p=[none|quarantine|reject]`: The policy for messages failing authentication (mandatory).
        *   `p=none`: Monitor mode. Don't take action, but send reports. Ideal for initial deployment.
        *   `p=quarantine`: Mark messages failing authentication as spam or put them in quarantine.
        *   `p=reject`: Reject messages failing authentication. This is the strongest enforcement.
    *   `rua=mailto:email@yourdomain.com`: Email address(es) to receive aggregate DMARC reports (optional, highly recommended). These are XML reports summarizing authentication results.
    *   `ruf=mailto:email@yourdomain.com`: Email address(es) to receive forensic/failure reports (optional, less commonly supported due to privacy concerns).
    *   `pct=[1-100]`: Percentage of messages to apply the policy to (optional, default 100). Useful for gradual rollout.
    *   `adkim=[s|r]`: Alignment mode for DKIM (`s`=strict, `r`=relaxed).
    *   `aspf=[s|r]`: Alignment mode for SPF (`s`=strict, `r`=relaxed).
    *   `sp=[none|quarantine|reject]`: Policy for subdomains (optional, defaults to `p`'s policy).

    **Example:**
    ```
    v=DMARC1; p=reject; rua=mailto:dmarc-reports@example.com; pct=100; adkim=s; aspf=s
    ```
3.  **Publish the DMARC Record:** Add this TXT record to your domain's DNS settings. The record name **must** be `_dmarc.yourdomain.com`.

4.  **Gradual Deployment is Key:** Never jump straight to `p=reject`. A recommended DMARC rollout strategy is:
    *   **Phase 1 (Monitoring):** `p=none` with `rua` reports. Analyze reports to ensure legitimate mail passes SPF/DKIM.
    *   **Phase 2 (Quarantine):** Gradually increase `pct` with `p=quarantine`. Continue monitoring reports for false positives.
    *   **Phase 3 (Enforcement):** Move to `p=reject` with `pct=100`, making sure your legitimate emails are fully aligned.

    DMARC reports (XML format) can be complex to read. Utilize online tools and dashboards provided by your smart host or third-party services to interpret them effectively.

## BIMI (Brand Indicators for Message Identification): Visual Trust

**BIMI** is a newer standard that allows you to display your brand's logo next to your email in supporting inboxes (e.g., Gmail, Yahoo Mail). It reinforces brand recognition and enhances trust, which can significantly boost user engagement.

### Technical Implementation:

1.  **Prerequisites: Strong SPF, DKIM, DMARC:** BIMI requires strict adherence to SPF, DKIM, and DMARC. Your DMARC policy must be robust (e.g., `p=quarantine` or `p=reject`) and apply to your entire domain, not just subdomains.
2.  **Certified Logo (VMC):** Your logo must be a registered trademark, and you need to obtain a **Verified Mark Certificate (VMC)** from an authorized certificate authority (e.g., Digicert, Entrust). The VMC links your registered trademark to your domain.
    *   The logo itself must be an SVG file (square, solid background).
3.  **Host Logo and Certificate:** Upload your SVG logo and the PEM-formatted VMC file to a secure, publicly accessible location on your server (accessible via HTTPS).
4.  **Construct the BIMI Record:** Create a TXT record containing a URL to your SVG logo and VMC.

    *   `_bimi.yourdomain.com`: The record name.
    *   `v=BIMI1; l=https://www.example.com/logo.svg; a=https://www.example.com/certificate.pem`: The value string.
        *   `l`: URL to your SVG logo.
        *   `a`: URL to your VMC file.

    **Example:**
    ```
    default._bimi.example.com IN TXT "v=BIMI1;l=https://cdn.example.com/logo.svg;a=https://cdn.example.com/certs/example-vmc.pem"
    ```
5.  **Publish the BIMI Record:** Add this TXT record to your domain's DNS settings.

### Benefits for Developers:

Implementing BIMI, while requiring more upfront effort and cost (for trademark registration and VMC), provides a verifiable visual cue that your emails are authentic, increasing engagement and further solidifying your sender reputation.

## Conclusion

Mastering the authentication stack is paramount for any developer involved in email delivery. SPF, DKIM, DMARC, and BIMI, when correctly configured, create a robust defense against malicious actors and significantly improve the likelihood of your legitimate emails reaching the inbox. While the initial setup may seem complex, the long-term benefits in deliverability and sender trust are invaluable. Always remember to validate your configurations and deploy changes gradually, especially with DMARC.