# Sources & Further Reading

This document is an original synthesis, written to stand on its own — it does not reproduce text from any of the sources below. They're listed here because they're genuinely useful further reading for anyone going deeper on a specific topic this document touches, and because some of the terminology and framing used in Part 17 and Part 19 in particular was informed by the broader public discussion these sources are part of.

## On "AI Slop" in Design & UI

- [AI Design Slop: 16 Patterns That Out Your App as Vibe-Coded — Developers Digest](https://www.developersdigest.tech/blog/ai-design-slop-and-how-to-spot-it)
- [Why Your AI Keeps Building the Same Purple Gradient Website](https://prg.sh/ramblings/Why-Your-AI-Keeps-Building-the-Same-Purple-Gradient-Website)
- [AI Design Slop: Why Every AI-Built Interface Looks the Same (And How to Fix It) — Mohit Phogat](https://mohitphogat.medium.com/ai-design-slop-why-every-ai-built-interface-looks-the-same-and-how-to-fix-it-bf874e0b470c)
- [AI Slop Fonts and Gradients: The Tells That Give Away AI Design — 925 Studios](https://www.925studios.co/blog/ai-slop-design-tells)
- [AI Slop Design: Why AI-Generated UI Looks Generic (Fix Guide) — VibeCodeKit](https://vibecodekit.dev/ai-slop-design)
- [Unslop UI: Kill the AI Design Tells — Claude Code Playbooks](https://www.claudecodehq.com/playbooks/unslop-ui)
- [Claude Code UI Slop Is Killing Your Front-End Taste — Productive Tech Talk](https://productivetechtalk.com/2026/04/16/claude-code-ui-slop-is-killing-your-frontend-taste/)
- [The Anti-AI-Slop Design Skill: How Hallmark Fixes Generic AI UI — Rohit Raj](https://rohitraj.tech/hi/notes/anti-ai-slop-design-skill-hallmark-guide-2026)
- [The End of "AI Slop": How UI/UX Pro Max Is Solving the Design Crisis in AI-Generated Code — Abhinav Dobhal](https://medium.com/@abhinav.dobhal/the-end-of-ai-slop-how-ui-ux-pro-max-is-solving-the-design-crisis-in-ai-generated-code-bbc23995f0e0)

## On "Anti-Slop" Rulesets for Coding Agents

- [github.com/miqdadbadjuber/anti-slop](https://github.com/miqdadbadjuber/anti-slop) — rules for an AI coding agent to filter generic AI-generated UI, text, and code.
- [github.com/peakoss/anti-slop](https://github.com/peakoss/anti-slop) — a GitHub Action that detects and closes low-quality/AI-slop pull requests.
- [github.com/kjmagnan1s/anti-slop](https://github.com/kjmagnan1s/anti-slop) — a tool for detecting, rewriting, and filtering AI-slop writing.
- [github.com/realrossmanngroup/no_ai_slop_writing_rules](https://github.com/realrossmanngroup/no_ai_slop_writing_rules) — a portable CLAUDE.md and skill set for keeping AI-assisted writing in a specific human voice.
- [GitHub Topics: anti-slop](https://github.com/topics/anti-slop) / [ai-slop](https://github.com/topics/ai-slop) / [anti-ai-slop](https://github.com/topics/anti-ai-slop) / [ai-slop-detection](https://github.com/topics/ai-slop-detection) — browsable indexes of the broader open-source ecosystem around this topic.

## On Agent Instruction Files (AGENTS.md / CLAUDE.md / Cursor Rules)

- [How to Build Your AGENTS.md — Augment Code](https://www.augmentcode.com/guides/how-to-build-agents-md)
- [CLAUDE.md, AGENTS.md & Copilot Instructions: Configure Every AI Coding Assistant — DeployHQ](https://www.deployhq.com/blog/ai-coding-config-files-guide)
- [CLAUDE.md, AGENTS.md, and Every AI Config File Explained — DEV Community](https://dev.to/deployhq/claudemd-agentsmd-and-every-ai-config-file-explained-4pde)
- [AGENTS.md Spec: Recommended Sections + AGENTS.md vs CLAUDE.md vs .cursorrules — MorphLLM](https://www.morphllm.com/agents-md-guide)
- [AGENTS.md vs .cursorrules vs Claude Skills — BuildBetter](https://blog.buildbetter.ai/agents-md-vs-cursorrules-vs-claude-skills-2026-comparison/)
- [CLAUDE.md vs AGENTS.md vs Cursor Rules — GetUnblocked](https://getunblocked.com/blog/claude-md-vs-agents-md-vs-cursor-rules/)
- [Writing a Good CLAUDE.md — HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [AGENTS.md vs CLAUDE.md vs Cursor Rules — CodersEra](https://codersera.com/blog/agents-md-vs-claude-md-vs-cursor-rules-comparison-2026/)
- [AGENTS.md for Codex and AI Coding Agents — BSWEN](https://docs.bswen.com/blog/2026-08-24-how-to-write-agents-md/)

## Formal Standards Referenced in Part 20

- [PEP 8 — Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [Google Style Guides](https://google.github.io/styleguide/)
- [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)
- [PSR-12: Extended Coding Style — PHP-FIG](https://www.php-fig.org/psr/psr-12/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [Kotlin Coding Conventions](https://kotlinlang.org/docs/coding-conventions.html)
- [Web Content Accessibility Guidelines (WCAG) 2.1 — W3C](https://www.w3.org/TR/WCAG21/)
- [WAI-ARIA Overview — W3C](https://www.w3.org/WAI/standards-guidelines/aria/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Keep a Changelog](https://keepachangelog.com/)
- [The Twelve-Factor App](https://12factor.net/)
- [The OpenAPI Specification](https://spec.openapis.org/)

## A Note on Attribution

The rules and explanations in this document are original writing, produced by synthesizing widely-shared, publicly-taught engineering and design knowledge — not transcribed or paraphrased line-by-line from any single source above. Where a source materially shaped how a section of this document frames its topic (most notably the design-slop vocabulary in Part 17, and the agent-instruction-file conventions in Part 19), that's noted here rather than through inline citation, in keeping with this being a reference document meant to be used as a standalone whole. Corrections, additions, and disagreements are welcome as issues or pull requests against this repository.
