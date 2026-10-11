# minimize-vuln-report

A Claude Code plugin that condenses a vulnerability writeup into a minimal report with three parts:

- a short description of the issue
- a minimal proof of concept
- a minimal patch

It accepts security advisories (GHSA, OSV, CVE pages), bug bounty and pentest reports, SARIF/SAST results, fuzzer and sanitizer crash reports, AI-generated findings, [Scrutineer](https://github.com/alpha-omega-security/scrutineer) disclosure bundles, issue threads, and pasted text.

## Verification

PoCs and patches are tested inside a Docker container, never on your machine. If you don't name an image or container in your request, Claude asks for one and suggests a matching local image or a standard one (e.g. `python:3.12-slim`). Without a container, you still get the report, with the PoC and patch marked as untested. PoC payloads are kept harmless (a marker file instead of a reverse shell) and are only ever pointed at local code.

## Install

```
/plugin marketplace add scovetta/minimize-vuln-report
/plugin install minimize-vuln-report@minimize-vuln-report
```

You can also pass a local path or a git URL to `/plugin marketplace add`.

## Use

Point Claude at a report (a file, a directory, a URL, or pasted text) and ask for the minimal report, optionally naming a Docker image for testing, or run:

```
/minimize-vuln-report:minimize-vuln-report <path-or-url>
```

The report is written as `<project>-<id>-minimal.md`.

## License

MIT
