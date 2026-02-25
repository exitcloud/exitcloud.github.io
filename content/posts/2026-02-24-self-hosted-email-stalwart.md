---
title: "Self-Hosted Email with Stalwart: Is It Worth the Pain?"
date: 2026-02-24T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "self-hosted"
  - "email"
---

# Self-Hosted Email with Stalwart: Is It Worth the Pain?

I'm going to be honest with you. Self-hosting email is the hardest thing on this list. Harder than [Kubernetes](https://kubernetes.io/). Harder than setting up CI/CD from scratch. Not because the software is complex — it's actually gotten remarkably good — but because the email ecosystem is a hostile, gatekept mess that actively punishes newcomers.

I've done it. I'm running it right now. And I'm going to tell you exactly what it takes so you can decide if it's worth it for you.

Spoiler: for most people, it's not. But for some of us, it absolutely is.

## Why Self-Host Email at All?

Three reasons:

1. **Privacy.** Google reads your email. Microsoft reads your email. They'll tell you it's for "improving services." You know what it's actually for.

2. **Control.** I want `me@mydomain.com`, and I want to own the entire pipeline. I don't want to wake up one day to find my account suspended because some algorithm flagged me.

3. **The challenge.** Okay, fine, I'm a nerd and I wanted to see if I could pull it off. Sue me.

If none of those resonate, just go use [Fastmail](https://www.fastmail.com/) ($5/mo) or [Migadu](https://www.migadu.com/) ($9/mo). Seriously. Both are excellent. Migadu in particular is privacy-focused and lets you use your own domain. No shame in that choice. You'll save yourself a lot of headaches.

Still here? Alright. Let's do this.

## Enter Stalwart

[Stalwart](https://stalw.art/) is a relatively new mail server written in Rust. It handles SMTP, IMAP, and JMAP in a single binary. No [Postfix](https://www.postfix.org/) + [Dovecot](https://www.dovecot.org/) + [OpenDKIM](http://www.opendkim.org/) + [Rspamd](https://rspamd.com/) Frankenstein stack. One binary. One config. One process.

That alone sold me. The traditional self-hosted email setup requires you to configure and maintain four or five separate daemons that all need to talk to each other correctly. Miss one setting and your mail either doesn't get delivered or gets marked as spam. Stalwart collapses all of that into one thing.

It's also actively maintained, has a web admin interface, and supports modern standards out of the box: JMAP (the JSON-based email protocol that's way better than IMAP), MTA-STS, DANE, ARC, and all the authentication stuff you need.

## The Docker Compose Setup

Here's my actual production setup, slightly anonymized:

```yaml
services:
  stalwart:
    image: stalwartlabs/mail-server:latest
    container_name: stalwart-mail
    restart: unless-stopped
    ports:
      - "25:25"       # SMTP
      - "465:465"     # SMTP submission (implicit TLS)
      - "587:587"     # SMTP submission (STARTTLS)
      - "993:993"     # IMAPS
      - "4190:4190"   # ManageSieve
      - "8080:8080"   # Web admin + JMAP
    volumes:
      - stalwart-data:/opt/stalwart-mail
    environment:
      - STALWART_ADMIN_PASSWORD=changethis
    hostname: mail.example.com

volumes:
  stalwart-data:
```

That's the whole thing. One container. Run `docker compose up -d` and you've got a mail server running.

But — and this is the big but — having the software running is maybe 20% of the battle. The other 80% is making sure the rest of the internet will actually accept mail from your server.

## The DNS Gauntlet

Here's where it gets real. You need to set up a bunch of DNS records, and every single one of them matters.

### Reverse DNS (PTR Record)

Your VPS's IP address needs a PTR record that points back to your mail hostname. This is set in your VPS provider's control panel, not in your DNS provider.

```
49.12.45.101 -> mail.example.com
```

Most providers ([Hetzner](https://www.hetzner.com/), [DigitalOcean](https://www.digitalocean.com/), [Vultr](https://www.vultr.com/)) let you set this in the dashboard. If your PTR record doesn't match your mail hostname, [Gmail](https://mail.google.com/) and [Outlook](https://outlook.com/) will reject your mail outright. Not spam-folder it. *Reject* it.

### MX Record

```
example.com.    MX    10    mail.example.com.
```

This tells the world "send mail for @example.com to this server."

### SPF

```
example.com.    TXT    "v=spf1 mx a ip4:49.12.45.101 -all"
```

SPF says "only these IPs are authorized to send mail for my domain." The `-all` at the end means "reject anything else." Use `-all`, not `~all`. Be strict.

### DKIM

Stalwart can generate DKIM keys for you through the admin interface. You'll get a DNS record that looks like:

```
default._domainkey.example.com.    TXT    "v=DKIM1; k=ed25519; p=BASE64KEYHERE"
```

DKIM cryptographically signs every outgoing email so the recipient can verify it wasn't tampered with and actually came from your server. Without it, you're going straight to spam.

Stalwart also supports dual DKIM signing (RSA + ed25519), which I recommend. Some older mail servers only understand RSA, while newer ones prefer ed25519.

### DMARC

```
_dmarc.example.com.    TXT    "v=DMARC1; p=reject; rua=mailto:dmarc@example.com; adkim=s; aspf=s"
```

DMARC ties SPF and DKIM together and tells receiving servers what to do when either check fails. `p=reject` means "if it fails, throw it away." Start with `p=none` while you're testing, then tighten to `p=quarantine`, then `p=reject` once you're confident everything is working.

### MTA-STS

This one's optional but increasingly expected. It tells sending servers to use TLS when delivering mail to you. You need a policy file hosted at `https://mta-sts.example.com/.well-known/mta-sts.txt`:

```
version: STSv1
mode: enforce
mx: mail.example.com
max_age: 604800
```

And a DNS record:

```
_mta-sts.example.com.    TXT    "v=STSv1; id=20260224"
```

## The Deliverability Problem

Here's the honest truth that nobody in the self-hosted email community likes to talk about: the big providers don't want you sending email from your own server.

Gmail, Outlook, [Yahoo Mail](https://mail.yahoo.com/) — they all maintain internal reputation scores for sending IPs. A brand new VPS IP has zero reputation. Zero reputation means your mail goes to spam. Or gets silently dropped. Or gets deferred for hours.

This is the single biggest challenge of self-hosted email, and there's no quick fix.

What you can do:

1. **Check your IP against blocklists.** Use [MXToolbox](https://mxtoolbox.com/blacklists.aspx) to check before you even start. If your IP is on a blocklist, request a new one from your VPS provider.

2. **Start slow.** Don't send 500 emails on day one. Send a few test messages to your own Gmail, Outlook, and Yahoo accounts. Check headers to see if you're passing SPF, DKIM, and DMARC.

3. **Register with postmaster tools.** [Google Postmaster Tools](https://postmaster.google.com/) and [Microsoft SNDS](https://sendersupport.olc.protection.outlook.com/snds/) give you visibility into how they're treating your mail.

4. **Be patient.** Building IP reputation takes weeks to months of consistent, legitimate email sending. There's no shortcut.

5. **Use a relay for transactional mail.** This is my actual recommendation: use your self-hosted server for personal/business email, but route transactional email (password resets, notifications) through a dedicated service like [Amazon SES](https://aws.amazon.com/ses/) ($0.10/1000 emails) or [Postmark](https://postmarkapp.com/). These services have established IP reputation and high deliverability. Configure Stalwart to relay outbound mail from specific addresses through their SMTP relay.

## The Web Interface

Stalwart ships with a web admin panel at port 8080. It handles user management, domain configuration, DKIM key generation, queue management, and log viewing. It's genuinely good.

For webmail, you'll want a separate client. I use [Roundcube](https://roundcube.net/) in a Docker container pointed at Stalwart's IMAP:

```yaml
  roundcube:
    image: roundcube/roundcubemail:latest
    restart: unless-stopped
    environment:
      - ROUNDCUBEMAIL_DEFAULT_HOST=ssl://stalwart-mail
      - ROUNDCUBEMAIL_DEFAULT_PORT=993
      - ROUNDCUBEMAIL_SMTP_SERVER=tls://stalwart-mail
      - ROUNDCUBEMAIL_SMTP_PORT=587
    depends_on:
      - stalwart
```

Or just use [Thunderbird](https://www.thunderbird.net/). Or [Apple Mail](https://support.apple.com/mail). Or any IMAP client. The whole point is that it's standard protocols.

## When It Makes Sense

Self-hosted email makes sense if:

- You care deeply about privacy and data sovereignty
- You have a custom domain and want full control over it
- You're comfortable with DNS and don't mind the initial setup time
- You accept that deliverability requires ongoing attention
- You enjoy the technical challenge (seriously, this matters)

## When to Just Use Fastmail/Migadu

Use a hosted provider if:

- Email is critical to your business and you can't afford deliverability issues
- You don't want to think about DNS records, IP reputation, or spam filtering
- You need guaranteed uptime without being on-call for your mail server
- You send a lot of transactional email (use a dedicated service for this anyway)
- You value your weekends

I'm running self-hosted email because I enjoy the control and I've accepted the tradeoffs. My personal email goes through Stalwart. My business transactional email goes through [Postmark](https://postmarkapp.com/). That split works perfectly.

If your time is worth more than $5/mo (it is), and you don't have a specific reason to self-host... just use Fastmail. You'll get your weekends back.

## Testing Your Setup

Before you declare victory, run your setup through these tools:

```bash
# Check all DNS records at once
dig +short MX example.com
dig +short TXT example.com
dig +short TXT default._domainkey.example.com
dig +short TXT _dmarc.example.com

# Send a test email and check headers
# Look for "spf=pass", "dkim=pass", "dmarc=pass"
```

Online tools to verify everything is working:

- [mail-tester.com](https://www.mail-tester.com/) — Send an email to their test address, get a score out of 10
- [learndmarc.com](https://www.learndmarc.com/) — Interactive DMARC debugger
- [mxtoolbox.com](https://mxtoolbox.com/) — DNS record checker and blocklist scanner

You want a 10/10 on mail-tester.com before you start using the server for real mail. Anything less and you need to fix something.

Ship it. Or don't. I won't judge.

## Cheat Sheet

| Tool / Concept | What It Does | Link |
|---|---|---|
| **Stalwart** | All-in-one mail server in Rust: SMTP, IMAP, JMAP in a single binary | [stalw.art](https://stalw.art/) |
| **SPF** | DNS TXT record declaring which IPs can send mail for your domain | - |
| **DKIM** | Cryptographic signature on outgoing mail proving authenticity | - |
| **DMARC** | Policy telling receivers what to do when SPF/DKIM fails. Start with `p=none`, end at `p=reject` | - |
| **PTR / Reverse DNS** | IP-to-hostname mapping set in your VPS panel. Must match your mail hostname | - |
| **MTA-STS** | Enforces TLS for inbound mail delivery. Hosted policy file + DNS record | - |
| **Roundcube** | Open-source webmail client. Point it at Stalwart's IMAP/SMTP | [roundcube.net](https://roundcube.net/) |
| **mail-tester.com** | Send a test email, get a deliverability score out of 10. Aim for 10/10 | [mail-tester.com](https://www.mail-tester.com/) |
| **MXToolbox** | Check DNS records and scan IP against blocklists before sending | [mxtoolbox.com](https://mxtoolbox.com/) |
| **learndmarc.com** | Interactive tool for debugging DMARC, SPF, and DKIM alignment | [learndmarc.com](https://www.learndmarc.com/) |
| **Google Postmaster Tools** | See how Gmail is treating your mail — reputation, spam rate, auth results | [postmaster.google.com](https://postmaster.google.com/) |
| **Postmark** | Transactional email relay with great deliverability. Use for app notifications | [postmarkapp.com](https://postmarkapp.com/) |
| **Amazon SES** | Cheap transactional email relay ($0.10/1000 emails) for outbound app mail | [aws.amazon.com/ses](https://aws.amazon.com/ses/) |
| **Fastmail** | Hosted email provider with custom domain support. $5/mo, just works | [fastmail.com](https://www.fastmail.com/) |
| **Migadu** | Privacy-focused hosted email. $9/mo, supports custom domains | [migadu.com](https://www.migadu.com/) |
| **Dual DKIM signing** | Sign with both RSA and ed25519 for maximum compatibility | - |
| `dig +short TXT` | Quick CLI check for SPF, DKIM, and DMARC DNS records | - |
