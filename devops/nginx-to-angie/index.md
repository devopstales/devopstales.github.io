---
title: Nginx, F5 Acquisition, and the Rise of Angie: Why You Should Consider Switching
url: https://devopstales.github.io/devops/nginx-to-angie/
date: 2026-03-27
keywords: nginx, angie, web server, F5 acquisition, open source, reverse proxy, load balancer
---


If you've been working with web servers and reverse proxies for the past decade, chances are you've used **nginx**. It's been the backbone of countless web infrastructures, powering everything from small personal blogs to some of the busiest websites on the planet. But recent developments in the nginx ecosystem have led to an interesting fork called **Angie** — and it might be worth your attention.

In this post, I'll walk through the history of nginx, what happened after the F5 acquisition, why Angie was created, and why you might want to consider making the switch.

<!--more-->

## The Rise of Nginx

Nginx was created by **Igor Sysoev** in 2004, with the first public release in 2005. It was designed to solve the **C10K problem** — handling 10,000 concurrent connections efficiently. Unlike Apache's process-based model, nginx used an **event-driven, asynchronous architecture** that could handle massive concurrency with minimal memory footprint.

Key innovations that made nginx successful:

- **Event-driven architecture** — No thread-per-connection overhead
- **Reverse proxy capabilities** — Excellent load balancing and proxy features
- **Static file serving** — Blazing fast static content delivery
- **Low memory usage** — Efficient resource utilization
- **Configuration simplicity** — Clean, declarative configuration syntax

By 2010, nginx was powering some of the world's busiest websites including Netflix, WordPress.com, and GitHub. It became the de facto standard for high-performance web serving and reverse proxying.

## The F5 Acquisition (2019)

In March 2019, **F5 Networks announced the acquisition of nginx for $670 million**. The deal closed in May 2019, bringing nginx under the umbrella of the established application delivery company.

F5's stated goals were clear:
- Bridge the gap between **NetOps and DevOps**
- Expand into **multi-cloud application services**
- Leverage nginx's open-source community for enterprise offerings

Initially, the acquisition seemed benign. Nginx continued to operate relatively independently, and the open-source version remained available under the BSD license. However, concerns began to emerge within the community about the long-term direction of the project.

### Community Concerns

Over time, several issues surfaced:

1. **Corporate control over open-source development** — Decisions increasingly driven by commercial interests rather than community needs
2. **Security policy changes** — Disagreements over how security vulnerabilities should be handled and disclosed
3. **Resource allocation** — Core developers leaving or being reassigned to commercial projects
4. **Geopolitical complications** — F5's exit from Russia following the Ukraine invasion left some nginx developers in limbo

The writing was on the wall: nginx was no longer the community-driven project it once was.

## Enter Angie: The Fork

In 2024, **Angie** emerged as a fork of nginx, created by **former nginx core developers** who had worked on the original project since 2011. The development team at Angie Software includes specialists who previously worked at F5 Networks.

### Why Angie Was Created

The motivations behind Angie were multifaceted:

**1. Developer-Led Development**

Angie is being built by developers who understand the codebase intimately — the same people who wrote much of nginx's core functionality. They wanted to return to a model where technical decisions are made by engineers, not corporate stakeholders.

**2. Addressing Architectural Limitations**

The Angie team identified fundamental shortcomings in nginx's architecture, particularly around:
- **HTTP/3 implementation** — nginx's approach had limitations that Angie aims to solve
- **Modern feature requirements** — Features like persistent caching and real-time monitoring weren't priorities for F5's nginx

**3. True Open-Source Spirit**

Angie is released under the **free BSD license**, maintaining the open-source ethos that made nginx popular in the first place. The project is committed to community-driven development without corporate interference.

**4. Continuity for Affected Developers**

Some of the Angie team were nginx developers who were left without support when F5 restructured operations in certain regions. Angie provided a path to continue their work.

## Why You Should Switch to Angie

Now for the important question: **should you switch from nginx to Angie?** Here are compelling reasons to consider making the move:

### 1. Drop-in Replacement

Angie is **fully backward compatible** with nginx configurations. Your existing nginx configuration files work with Angie without modification. This means:

- No configuration rewrites
- No staff retraining
- Minimal migration risk
- Zero implementation costs

You can literally swap the binaries and restart.

### 2. Enhanced Features Out of the Box

Angie includes features that either don't exist in nginx or require the commercial nginx Plus version:

**Cache Index Persistence**
- Save shared memory zones to disk
- Instant cache loading on restart
- No cache loader process needed
- Faster restarts with warm caches

**Advanced Load Balancing**
- `backup_switch` directive for permanent or timed backup switching
- Doesn't revert immediately when main servers recover
- Better control over failover behavior

**Modern Protocol Support**
- TLS 1.3 Early Data (0-RTT) in stream module
- Enhanced HTTP/3 implementation using eBPF
- Reduced connection latency

**Monitoring and Observability**
- Built-in metrics module
- "Busy" status in statistics API
- Custom statistics module (coming in 1.10)
- Prometheus export support

**ACME Module Improvements**
- `renew_on_load` parameter
- Better certificate management
- Preserves existing certificates with `enabled=off`

### 3. Docker Integration (Coming in 1.10)

Angie 1.10 introduces automatic upstream server list updates based on Docker container labels. This means:
- Automatic service discovery
- Real-time container start/stop detection
- Dynamic IP address management in upstreams
- No external service mesh required

### 4. Developer Expertise

The Angie team consists of:
- Core nginx developers since 2011
- Former F5 Networks specialists
- Engineers who built the features you rely on

This expertise translates to:
- Better bug fixes
- More thoughtful feature development
- Understanding of real-world deployment scenarios

### 5. Active Development

Angie has a clear release cadence:
- **Quarterly stable releases**
- Regular feature additions
- Responsive bug fixes
- Transparent roadmap

Compare this to nginx's increasingly unpredictable release schedule under F5.

### 6. Professional Support Available

For enterprise users, Angie PRO offers:
- Award-winning technical support
- Experience serving Fortune 500 companies
- Additional enterprise features:
  - Sticky sessions with external storage
  - UDP health checks
  - Docker module support
  - Enhanced API features

### 7. Future-Proof Architecture

Angie is addressing fundamental architectural issues:
- **HTTP/3 from the ground up** — Using eBPF for proper QUIC implementation
- **Modern monitoring** — Built-in observability features
- **Container-native** — Docker and Kubernetes integration
- **Extensible** — Custom modules and statistics

### 8. Community-First Approach

Unlike nginx under F5, Angie is committed to:
- Open development
- Community feedback
- Transparent decision-making
- BSD license freedom

## Migration: How to Switch

Switching to Angie is straightforward:

```bash
# 1. Install Angie (package manager varies by distribution)
# For example, on Debian/Ubuntu:
wget -qO - https://angie.software/keys/angie-keyring.gpg | gpg --dearmor | tee /usr/share/keyrings/angie-keyring.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/angie-keyring.gpg] https://angie.software/packages/debian stable main" | tee /etc/apt/sources.list.d/angie.list
apt update
apt install angie

# 2. Stop nginx
systemctl stop nginx

# 3. Your existing configuration works as-is
# Configuration files are in the same location
# /etc/nginx/ → /etc/angie/

# 4. Start Angie
systemctl start angie

# 5. Verify
systemctl status angie
angie -v
```

Test thoroughly in a staging environment first, but the transition should be seamless.

## Potential Considerations

Of course, no switch is without considerations:

**Ecosystem Maturity**
- Angie is newer, so third-party modules and community resources are smaller
- However, nginx compatibility means most resources still apply

**Long-Term Viability**
- Angie is a newer project, but the team's pedigree is strong
- The open-source model reduces single-points-of-failure

**Enterprise Support**
- If you have nginx Plus support contracts, evaluate Angie PRO
- Many organizations are already making the switch

## The Bottom Line

The nginx that conquered the web server world in the 2010s isn't the nginx of today. Under F5, it became a commercial product first and an open-source project second. **Angie represents a return to nginx's roots** — developer-driven, community-focused, and genuinely open source.

With full backward compatibility, enhanced features, and a team that literally wrote the code you're running, Angie is worth serious consideration for your next infrastructure refresh.

## Getting Started

Ready to try Angie?

- **Official Website**: [https://angie.software](https://angie.software)
- **Documentation**: [https://angie.software/en/docs/](https://angie.software/en/docs/)
- **GitHub**: [https://github.com/webserver-llc/angie](https://github.com/webserver-llc/angie)

Have you made the switch to Angie? Are you considering it? Share your thoughts in the comments below!

---

*Further Reading:*
- *F5 Acquires NGINX for $670M — TechCrunch, 2019*
- *Freenginx: A Fork of NGINX — The New Stack, 2024*
- *What's New in Angie 1.9 — Habr, 2025*

