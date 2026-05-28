# Security Policy

## Supported Versions

This repository is currently an initial scaffold. Until the first public release, treat all versions as pre-release.

## Reporting a Vulnerability

Report suspected vulnerabilities through GitHub Security Advisories when the public repository is available. You can also contact Seezo at security@seezo.io.

Do not open public GitHub issues for vulnerabilities. Do not include secrets, customer data, private security reports, or exploit details beyond what is needed to reproduce the issue safely.

## Public Repository Guardrails

This repository is intended to be public-safe from the first commit. Contributions must not include:

- internal Seezo prompts, evaluator logic, scoring rubrics, or proprietary security rules;
- customer names, evidence, questionnaires, reports, screenshots, logs, or private architecture details;
- private Seezo endpoints, tokens, API keys, OAuth secrets, credentials, or environment dumps;
- realistic exploit payloads tied to real customer systems.

Use synthetic examples and placeholders only. If a change needs private implementation details, keep those details in the remote Seezo MCP service or a private repository, not in this plugin.
