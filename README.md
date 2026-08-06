# Prism Gateway

The security-first Rust API gateway that auto-translates payloads, self-generates schemas, and visualizes pipelines—self-hosted or cloud-ready.

[![License: MIT OR Apache-2.0](https://img.shields.io/badge/License-MIT%20OR%20Apache--2.0-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.81+-orange.svg)](https://www.rust-lang.org/)

---

## What is Prism Gateway?

Prism Gateway is a tool that removes the friction between different data formats and security requirements in your API ecosystem. It is designed to be immediately useful without requiring deep configuration or prior expertise.

Instead of writing manual mapping code or juggling format-specific parsers, you deploy Prism as a single binary. It sits between your clients and your upstream services, inspecting incoming payloads and transforming them into the exact schema your backend expects. The translation supports JSON, TOML, YAML, XML, and extensible custom formats.

A built-in pattern-recognition component observes real request traffic and suggests clear, maintainable schemas for your routes. These suggestions appear directly in the dashboard—you can accept, modify, or discard them with one click, giving you full control without manual guesswork.

The project also includes a visual dashboard that operates like a design canvas. You can drag and drop route definitions, preview live payload transformations as they happen, and embed custom middleware code using an integrated editor. The preview updates instantly, allowing rapid iteration without redeploying.

## Security Philosophy

Security is embedded at every layer. The gateway enforces OWASP ESAPI-aligned controls including strict input validation, output encoding, and request-scoped access management. It continuously ingests automated vulnerability intelligence from CVE, GCVE, and EUVD feeds. When a known vulnerability affects one of your dependencies or upstream services, the system highlights the risk and proposes concrete remediation steps directly in the UI. All security guidance is generated before any transformation or forwarding occurs, ensuring that policy violations are blocked early.

Data privacy is a first-class concern. In its default mode, Prism operates entirely offline. All vulnerability databases and processing logic reside on your local machine or private network. No payload content, schema data, or traffic metadata is transmitted to external services unless you explicitly enable cloud mode.

## Key Features

- Universal payload translation between JSON, TOML, YAML, XML, and custom schemas
- Heuristic schema suggestion engine that learns from live traffic patterns
- Visual dashboard with drag-and-drop route design and live transformation preview
- Embedded Monaco editor for custom middleware and transformation scripts
- OWASP ESAPI-compliant validation, encoding, and access control
- Automated CVE, GCVE, and EUVD vulnerability feed integration
- Zero-trust data handling with full offline operation
- Single compiled Rust binary with no runtime dependencies
- Flexible deployment: bare-metal, container, or cloud (AWS/Cloudflare)

