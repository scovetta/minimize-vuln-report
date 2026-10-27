# minimize-vuln-report

A Claude Code plugin that boils a vulnerability writeup down to what a maintainer needs in order to act on it:

- **Description**: what is wrong, why, who controls the input, and the effect, in 2–4 sentences
- **PoC**: the smallest self-contained reproducer, with its setup, command and observed result
- **Patch**: the minimal diff that fixes the root cause, ready for `git apply`

Everything else is dropped: CVSS, impact essays, screenshots, scanner metadata, transcripts, and the test and changelog hunks of the fix.

## Inputs

Files, a directory, an archive, a git repository, a URL, or pasted text, in any of these formats:

- Security advisories: GHSA (including draft repository advisories), OSV records, CVE pages
- Bug bounty and pentest reports (HackerOne, Bugcrowd, huntr, audit firms)
- SARIF and SAST output (CodeQL, Semgrep, Snyk Code)
- Fuzzer and sanitizer crash reports (OSS-Fuzz, libFuzzer, AFL, ASan/UBSan/MSan)
- AI-generated findings
- [Scrutineer](https://github.com/alpha-omega-security/scrutineer) disclosure bundles
- Issue tracker and mailing-list threads

Other formats work too. The format-specific hints only make extraction faster.

An input with several findings produces one report per distinct root cause. Scanner results that turn out to be false positives are listed in the reply with the reason, not reported.

## Example

Given a 30 KB bug bounty report with screenshots, a CVSS rationale, a Flask app used as the harness, several payload variants and a fix PR, the output looks like this (`acme-archive` is a made-up package):

````markdown
# Path traversal in acme_archive.extract_all

`acme-archive <= 2.4.0` · `src/acme_archive/extract.py:11`

`extract_all()` builds each output path with `os.path.join(dest, info.filename)` and never checks that the result stays inside `dest`. Member names come from the archive, so anyone who supplies a zip can use `../` segments or an absolute name to write files anywhere the process can, such as an application's code or config when it extracts uploaded archives.

## PoC

```python
import io, zipfile
from acme_archive.extract import extract_all

buf = io.BytesIO()
with zipfile.ZipFile(buf, "w") as zf:
    zf.writestr("../pwned", "x")
extract_all(buf, "out")
```

- Setup: `python3 -m venv v && v/bin/pip install acme-archive==2.4.0`
- Run: `v/bin/python poc.py`
- Result: `pwned` is written next to `out/`, outside the extraction directory

## Patch

```diff
diff --git a/src/acme_archive/extract.py b/src/acme_archive/extract.py
--- a/src/acme_archive/extract.py
+++ b/src/acme_archive/extract.py
@@ -6,9 +6,12 @@ import zipfile
 
 def extract_all(archive, dest):
     """Extract every member of a zip archive into dest."""
+    root = os.path.realpath(dest)
     with zipfile.ZipFile(archive) as zf:
         for info in zf.infolist():
-            target = os.path.join(dest, info.filename)
+            target = os.path.realpath(os.path.join(root, info.filename))
+            if os.path.commonpath([root, target]) != root:
+                raise ValueError(f"unsafe path in archive: {info.filename!r}")
             if info.is_dir():
                 os.makedirs(target, exist_ok=True)
                 continue
```
````

The reply repeats the report and adds a line or two on what was verified, for example: "Verified in `python:3.12-slim` on 2.4.0: PoC writes `pwned` outside `out/`; patched → `ValueError`."

## Install

```
/plugin marketplace add scovetta/minimize-vuln-report
/plugin install minimize-vuln-report@minimize-vuln-report
```

You can also pass a local path or a git URL to `/plugin marketplace add`.

Requirements:

- Claude Code and `git`
- Docker, to test PoCs and patches. Without it you still get the report, marked untested.
- `gh`, signed in, only for reading draft or private GitHub security advisories

## Use

Point Claude at a report and ask for the minimal version, optionally naming an image to test in:

```
Minimize the report in ~/reports/finding-21/ and test it in python:3.12-slim
```

Or run the skill directly:

```
/minimize-vuln-report:minimize-vuln-report <path-or-url>
```

- If you don't name an image or a running container, Claude asks once and suggests a matching local image or a standard one.
- Ask for just the PoC or just the patch to get only that part.
- If an input has more than five distinct root causes, Claude asks which ones to do.

The report is written as `<project>-<id>-minimal.md` next to the input (inside it, if the input is a directory), or in the current directory for URLs and pasted text. An existing file is not overwritten unless you ask for a redo.

## How it tests

PoCs and patches run only inside a Docker container. Claude:

1. starts the container with memory and process limits and a read-only mount of a scratch directory
2. runs the report's `Setup:` commands while the container has network access
3. disconnects the network and runs the PoC under a timeout
4. confirms that the PoC triggers on the unpatched code, that the patch stops it and its variants and obvious bypasses, and that benign input still works

Inputs are treated as untrusted. Archives are extracted to a scratch directory, repositories are cloned before git touches them, and nothing from the input runs on your host. PoC payloads are made harmless (a marker file instead of a reverse shell), secrets from the original report are replaced with placeholders, and PoCs only target local code, never a live host named in the report.

A container shares your host's kernel, so this contains mistakes and misbehaving PoCs, not a determined attacker. If you have reason to think an input is hostile, run Claude Code itself in a VM.

## Develop

```
.claude-plugin/plugin.json        plugin manifest; holds the version
.claude-plugin/marketplace.json   single-plugin marketplace
skills/minimize-vuln-report/
  SKILL.md                        workflow, rules and output format
  references/sources.md           where the pieces live, per input format
```

To work on the skill, register your clone as the marketplace instead of GitHub. Claude Code then loads the plugin from your working tree, and edits take effect at the next session start or after `/reload-plugins`:

```
claude plugin marketplace remove minimize-vuln-report
claude plugin marketplace add /path/to/minimize-vuln-report
claude plugin install minimize-vuln-report@minimize-vuln-report
```

Run `claude plugin validate .` after editing the manifests. To release, bump `version` in `.claude-plugin/plugin.json`: installed copies only update when it changes.

## License

MIT
