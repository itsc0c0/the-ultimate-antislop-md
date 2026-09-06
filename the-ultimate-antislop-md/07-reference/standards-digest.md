# General Software Engineering Standards Digest

This document synthesizes and organizes widely-shared engineering and design judgment rather than inventing it from scratch. A number of formal, publicly maintained standards and widely-adopted style guides cover pieces of the same ground in far more depth than a single reference document can. This part is a map to those — what each one is for, and when to reach for the primary source instead of (or alongside) this document — rather than a restatement of their contents.

## Language & Framework Style Guides

- **PEP 8** — the Python community's official style guide, covering naming, layout, and idiomatic conventions for Python code. Part 5 of this document draws on the spirit of PEP 8 but PEP 8 itself is the authoritative, exhaustive source, especially for exact formatting conventions. Maintained at python.org.
- **The Google Style Guides** — a family of style guides covering Python, Java, C++, JavaScript, Go, Shell, HTML/CSS, and several other languages, published by Google and widely used as a reference well beyond Google itself, particularly for conventions this document treats more briefly (exact formatting, specific naming edge cases). Published at google.github.io/styleguide.
- **Airbnb's JavaScript Style Guide** (and its companion React/JSX guide) — one of the most widely adopted community style guides for JavaScript and React, covering formatting, idioms, and component conventions in detail. Useful as a concrete, opinionated reference when a project hasn't already settled its own JS/React conventions.
- **PSR-12** (and the broader PHP-FIG PSR family) — the PHP community's standard for coding style, extending the older PSR-1/PSR-2 standards. The authoritative source for PHP formatting conventions referenced in Part 8. Maintained by the PHP Framework Interop Group at php-fig.org.
- **Effective Go** and the Go team's own style guidance — the closest thing Go has to an official style guide, covering idiomatic patterns beyond what `gofmt` enforces mechanically. Published at go.dev.
- **The Rust API Guidelines** — a checklist-style reference for designing idiomatic, consistent public APIs in Rust crates, covering naming, trait implementations, and documentation conventions. Published under the Rust project at rust-lang.github.io.
- **The Ruby Style Guide** (community-maintained, often referred to as "the Ruby Style Guide" or by its `rubocop` tooling namesake) — a widely-used community reference for idiomatic Ruby formatting and conventions.
- **Swift API Design Guidelines** — Apple's official guidance for naming and designing Swift APIs, particularly influential for how Swift code reads at call sites. Published at swift.org.
- **The Kotlin Coding Conventions** — JetBrains' official style guide for idiomatic Kotlin, covering formatting and idiom choices specific to the language. Published at kotlinlang.org.
- **Microsoft's C# Coding Conventions and .NET API design guidelines** — Microsoft's official guidance for C# style and for designing consistent public .NET APIs. Published as part of the .NET documentation at learn.microsoft.com.

## Web Standards & Accessibility

- **WCAG (Web Content Accessibility Guidelines), currently version 2.1/2.2, with 3.0 in development** — the W3C's formal accessibility standard, defining specific, testable success criteria at A/AA/AAA conformance levels. Parts 9 and 18 of this document summarize WCAG-aligned practices at a high level; WCAG itself is the authoritative source for exact conformance criteria, and the one to consult for any compliance-driven accessibility work. Published by the W3C at w3.org/WAI/WCAG21.
- **WAI-ARIA** — the W3C's specification for Accessible Rich Internet Applications, defining the roles, states, and properties that make custom interactive components usable with assistive technology. The authoritative source for correct ARIA usage referenced in Parts 9 and 18.
- **HTML Living Standard** — the WHATWG's continuously updated specification for HTML, the authoritative source for semantic element usage referenced throughout the frontend-related parts of this document.

## Security

- **The OWASP Top 10** — the Open Web Application Security Project's regularly updated ranking of the most critical web application security risks, along with OWASP's broader library of cheat sheets covering specific vulnerability classes (injection, authentication, XSS, and more) in far more depth than Part 15 of this document. Published at owasp.org.
- **The OWASP Application Security Verification Standard (ASVS)** — a more detailed, checklist-oriented standard for verifying application security controls, useful as a rigorous checklist beyond OWASP's higher-level Top 10.
- **CWE (Common Weakness Enumeration)** — a formal, categorized taxonomy of software weakness types, maintained by MITRE, useful as a precise vocabulary for describing specific classes of security bugs.

## Process, Versioning & Collaboration

- **Conventional Commits** — a lightweight specification for structuring commit messages (`type(scope): description`) in a way that supports automated changelog generation and semantic version bumps. Part 12 of this document references the spirit of structured, meaningful commit messages; Conventional Commits is a specific, widely-adopted convention for doing so mechanically. Published at conventionalcommits.org.
- **Semantic Versioning (SemVer)** — the widely-adopted convention for version numbers (`MAJOR.MINOR.PATCH`) that communicate the nature of changes between releases. The authoritative reference for how to version a public package or API. Published at semver.org.
- **Keep a Changelog** — a convention for structuring a human-readable `CHANGELOG.md`, complementary to SemVer's machine-readable version numbers. Published at keepachangelog.com.
- **The Twelve-Factor App** — a methodology for building software-as-a-service applications, covering configuration, dependencies, logging, and deployment in a way that has heavily influenced modern cloud-native and containerized application conventions referenced in Part 12. Published at 12factor.net.

## API & Data Format Standards

- **The OpenAPI Specification** — the standard schema format for describing REST APIs in a machine-readable way, referenced in Part 10's API documentation guidance as the standard tool for the job. Published by the OpenAPI Initiative at spec.openapis.org.
- **RFC 7231 and related HTTP RFCs** — the IETF specifications defining HTTP methods, status codes, and semantics, the authoritative source for the HTTP-verb and status-code correctness referenced in Part 10.
- **ISO 8601** — the international standard for representing dates and times unambiguously, relevant wherever this document touches on date/time handling and serialization formats.
- **RFC 2119 / RFC 8174** — the IETF convention for using capitalized keywords (MUST, SHOULD, MAY, and their negatives) with precise, agreed-upon meanings in technical specifications — worth knowing as a vocabulary if writing formal specs or RFC-style internal documents, distinct from this document's own DO/DON'T convention.

## How to Use This Digest

None of the above standards are reproduced here — each is reachable at its official source, linked in Part 22. Where this document's guidance and a formal standard listed here overlap, treat the formal standard as authoritative on exact, checkable detail (a specific contrast ratio, a specific status code, a specific commit-message grammar), and treat this document as a broader map connecting that detail to the surrounding judgment calls — why the rule exists, what it's trying to prevent, and how it relates to adjacent concerns the formal standard doesn't cover on its own.
