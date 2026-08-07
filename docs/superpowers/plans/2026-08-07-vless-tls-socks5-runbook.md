# VLESS + TLS + SOCKS5 Runbook Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (- [ ]) syntax for tracking.

**Goal:** Publish a sanitized, reproducible engineering runbook as a Jekyll blog post.

**Architecture:** Add one source Markdown post on main; the existing deploy workflow builds Jekyll and publishes the generated site to gh-pages. Keep the guide self-contained, use placeholders for all production identifiers, and validate formatting plus a Jekyll build.

**Tech Stack:** Jekyll, al-folio, Markdown, Ruby/Bundler, Prettier.

## Global Constraints

- Create content only in \_posts/; do not edit generated files on gh-pages.
- Use the exact post filename \_posts/2026-08-07-vless-tls-socks5-runbook.md.
- Use only placeholders for domains, addresses, tokens, panel paths and credentials.
- State observed versions exactly: 3X-UI 3.6, Xray 26.7.28, Mihomo Meta 1.19.29.

---

### Task 1: Create the technical runbook post

**Files:**

- Create: \_posts/2026-08-07-vless-tls-socks5-runbook.md

**Interfaces:**

- Consumes: Jekyll post front matter and the existing main-to-gh-pages deployment workflow.
- Produces: A rendered blog post at the Jekyll-generated 2026 blog route.

- [ ] **Step 1: Add Jekyll front matter**

Use the post layout, the title “VLESS + TLS + SOCKS5 链式代理：可复现部署与排障 Runbook”, date 2026-08-07 15:00:00 +0800, and tags for VLESS, Xray, 3X-UI, Cloudflare, Clash, SOCKS5 and operations.

- [ ] **Step 2: Write the deployment and verification sections**

Include a version matrix, an ASCII flow diagram, Cloudflare/DNS prerequisites, TLS inbound requirements, a Clash profile-specific JavaScript template, service checks, and a SOCKS5 TCP/auth/connect test.

- [ ] **Step 3: Write failure interpretation and handoff sections**

Include YAML-versus-base64 subscription behavior, missing dialer node errors, REALITY authentication failures, source-IP-dependent residential proxy failures, provider endpoint rotation, and a secret-safe handoff checklist.

- [ ] **Step 4: Verify the post is sanitized**

Run:

```powershell
rg -n "77\\.73\\.|38\\.244\\.|168\\.143\\.|cv\\.chenhan|RvEFm|YDxZ|f5cdc|360225|CBFm" _posts/2026-08-07-vless-tls-socks5-runbook.md
```

Expected: no output.

### Task 2: Validate site source

**Files:**

- Test: \_posts/2026-08-07-vless-tls-socks5-runbook.md

**Interfaces:**

- Consumes: the completed post from Task 1.
- Produces: formatting and Jekyll build evidence.

- [ ] **Step 1: Run Prettier validation**

Run:

```powershell
npm run lint:prettier
```

Expected: the new Markdown file passes Prettier validation.

- [ ] **Step 2: Run the Jekyll build**

Run:

```powershell
bundle exec jekyll build
```

Expected: exit code 0 and a generated article under \_site/blog/2026/.

- [ ] **Step 3: Inspect repository status**

Run:

```powershell
git status --short
```

Expected: only the new blog post and the intentional planning records are listed.
