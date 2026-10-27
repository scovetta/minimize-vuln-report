# Where the pieces live, by source format

Read only the section for the format you have. Each one says where the facts, PoC and patch usually live, and what is usually noise.

## Contents
- Scrutineer disclosure bundle
- GitHub Security Advisory (GHSA) / OSV record / CVE page
- SARIF and SAST output (CodeQL, Semgrep, Snyk Code, etc.)
- Fuzzer and sanitizer reports (OSS-Fuzz, libFuzzer, AFL, ASan/UBSan/MSan)
- Bug bounty and pentest reports (HackerOne, Bugcrowd, huntr, audit firms)
- AI-generated findings
- Issue trackers, mailing lists, pasted writeups

## Scrutineer disclosure bundle

```
scrutineer-<repo>-finding-<id>-disclosure-<date>.md   # disclosure draft: Summary/Impact/Affected/PoC/Fix
scrutineer-finding-<id>-disclosure-<date>.tar.gz
  manifest.json   # repository, finding_id, title, severity, cwe
  osv.json        # affected ranges, details
  report.md       # Source, Trace, Validation, Rating, Suggested fix, Notes (verify/patch/critic)
  patch.diff      # tested fix, usually source + regression test
  poc/            # README.md (verbatim validation), probe files, transcripts, run.sh (often a stub)
```

- **Facts**: the `.md` Summary/Impact for wording. The `report.md` Location row, Trace and Source for code-level detail.
- **PoC**: the reproducer in `report.md`'s *verify* note (the verifier re-ran it), or the `.md` PoC section. `poc/` often holds one harness with a case per finding. Take only this finding's part.
- **Patch**: `patch.diff` (tested). The prose "Fix suggestion" can drift from it. It usually targets the development branch, which is right for an unfixed bug.
- **Checkout**: the worker may have had a pruned or shallow checkout. Clone upstream for history, the version range, or release line numbers.
- **Known quirks**:
  - The OSV `affected` block can name the wrong ecosystem. Use the coordinate from the project's own build file.
  - "Affected: all releases" is often a guess.
  - Probe files may need renaming to match their public class before they compile.
- **Noise**: Rating, CVSS, Prior art, Reach, Notes, Communications, transcripts, manifest metadata.

## GitHub Security Advisory (GHSA) / OSV record / CVE page

- **Fetch structured data first**: `https://api.osv.dev/v1/vulns/<ID>` returns JSON for GHSA, CVE and other OSV IDs. `gh api /advisories/<GHSA-ID>` returns GitHub's record.
- **Facts**: `summary`/`details`, or the advisory body. Affected ranges come from `affected[].ranges` / `versions`, or the "Affected versions" field.
- **PoC**: often absent, or vague ("a crafted input can..."). Look at the `references` for a report, issue, or test added by the fix. The fix's regression test is frequently the best PoC seed, because it contains a triggering input.
- **Patch**: references of type `FIX`, or links to commits/PRs. In OSV, a `GIT` range's `fixed` event is the fix commit. Fetch the commit diff (`https://github.com/<o>/<r>/commit/<sha>.diff`), then strip tests, changelogs and version bumps.
- **No fix link** (common; many advisories only cite NVD and mailing lists): look between the last affected and the first fixed release, restricted to the defect file, e.g. `git log --oneline <last-affected-tag>..<first-fixed-tag> -- <file>`. Without a clone, use the GitHub compare view (`/compare/<a>...<b>`) or the commits API with `path=`. You can also search commit messages for the CVE, GHSA or tracker ID (e.g. `TEXT-123`).
- **Fix doesn't apply to the release**: the fix commit often lands on a branch that has moved on since the last release. Write the equivalent change against the latest affected release instead (see SKILL.md).
- **Noise**: CVSS, CWE text, credits, timeline, aliases, database-specific fields.

## SARIF and SAST output

- **Facts**: in `runs[].results[]`:
  - `ruleId` and `message.text` give the bug class.
  - `locations[].physicalLocation` (artifact URI + region) gives the sink.
  - `codeFlows[].threadFlows[].locations[]` is the source→sink trace. Use it to state who controls the input.
- **Semgrep JSON**: Semgrep's own JSON has the same pieces under `results[]`: `check_id`, `path`, `start.line`, `extra.message`.
- **Report name**: a rule ID names the check, not the finding, so use a short slug in the report's filename.
- **Dedupe**: many results often share one root cause, e.g. one unsafe helper reached from several call sites. Group by root cause, not by result.
- **PoC**: almost never supplied. Build it from the code flow: find the public entry point at the flow's source, then pass a value that reaches the sink.
- **False positives**: SAST results often are false positives, e.g. the value is sanitized on the way, the path is unreachable, or the hit is in test code. Results that share a sink can split, with some real and some sanitized. If you can't trigger a result and the code shows why, tell the user it is a false positive and why, in one line, instead of writing a report.
- **Patch**: `fixes[]` if present (rare), or autofix suggestions. Otherwise write it. The flagged sink is not always the defect site: fix where the untrusted value is mishandled, the same way the code already handles its safe cases (e.g. a sibling call that already quotes its input).
- **Noise**: rule help text and its example payloads, tags, precision/severity metadata, taxonomies, tool/driver info, fingerprints.

## Fuzzer and sanitizer reports

- **Facts**:
  - The sanitizer's error line gives the bug class and access, e.g. `heap-buffer-overflow ... WRITE of size 200`.
  - The first in-project stack frame is usually where the symptom happens. The defect can sit earlier, e.g. in a length check that let a bad value through.
  - For use-after-free and overflows, the allocation and free stacks matter.
  - Name the underlying class in the title (buffer overflow, use-after-free). Heap vs stack often depends only on how the harness allocated the object.
- **PoC**: a generator for the crashing input, plus the harness or binary invocation.
  - If the input format isn't clear from the code, minimize first to learn which bytes matter: OSS-Fuzz's "Minimized Testcase", libFuzzer's `-minimize_crash=1 -runs=10000` (writes `minimized-from-*`), or `afl-tmin`. Then write a structured generator that shows the format (length fields, magic bytes) instead of transcribing fuzzer bytes. Use `printf` or `python3 -c`, which nearly every image has.
  - Reproducing the same bug class in the same in-project frame is enough. Sizes needn't match the filed crash, though matching them helps the maintainer recognize it.
  - Build the harness the way the project does (its fuzz build target, or `clang -g -fsanitize=fuzzer,address ...`), and put that build in `Setup:`. Check that it uses the compiler you expect: make's built-in `CC=cc` beats a Makefile's `CC ?= clang`, so pass `CC=clang` explicitly. Don't verify with OSS-Fuzz's `infra/helper.py`, even for a project integrated into OSS-Fuzz: it builds and runs the project in containers it starts with `--privileged`.
  - `Result:` is the error class, the access (READ/WRITE and size), and the first in-project frame. The `SUMMARY:` line often names an interceptor such as `__asan_memcpy` instead, along with build-specific paths.
  - Sanitizer options change the outcome: `ASAN_OPTIONS=halt_on_error=0` lets a crashing process continue and exit 0, and breaks `-minimize_crash`. Run with the defaults a reader would have.
- **Patch**: the fixing commit, if linked. OSS-Fuzz issues give a fixed revision range, and the fix is in it. Otherwise write a bounds or lifetime fix where the bad value gets through.
- **Noise**: full stack traces beyond the defect frame, shadow-byte dumps, register dumps, build/job metadata, regression ranges, ClusterFuzz boilerplate.

## Bug bounty and pentest reports

- **Facts**: "Summary" and "Steps to reproduce" hold the substance. "Impact" sections are usually inflated, so state the concrete effect the steps actually demonstrate. Pentest reports (often PDF) give each finding its own heading; take one finding at a time.
- **PoC**: the steps to reproduce, reduced to the single request or call that triggers it.
  - Strip auth/session setup that isn't needed for the bug. If the attacker must be logged in, say so in the description.
  - Replace out-of-band callbacks with a local listener.
  - Replace production hosts with a local instance, and never send requests to the live target. If there is no local instance (closed-source services), keep the reported request with the host replaced by a placeholder like `https://target.example`, and say in the reply that it was not run.
- **Patch**: rarely supplied. Write it against the open-source code if it is available. If the target is closed-source, leave the Patch section out and say so in the reply.
- **Noise**: screenshots, video links, CVSS, program/bounty metadata, triage chatter, greetings, executive summaries, methodology sections.

## AI-generated findings

- AI findings are especially prone to plausible but wrong paths, line numbers, function names and quoted code, even when the bug is real. Check them all against the source (see SKILL.md).
- Treat claimed impact and claimed reproduction output as claims. Only a run you can repeat, or code you have checked yourself, counts as evidence.
- Expect long trace and reasoning sections. The facts usually sit in a summary plus a sink location; the rest is noise.

## Issue trackers, mailing lists, pasted writeups

- **Facts**: the first report, as corrected by later maintainer comments. Later comments often narrow the root cause or the affected versions.
- **PoC**: the smallest snippet anyone in the thread confirmed. Maintainers often post a smaller one than the reporter did.
- **Patch**: a linked PR or commit, or a diff posted in the thread. Check that the patch that was actually merged matches the one discussed.
- **Noise**: discussion, +1s, workarounds, backport debate, duplicates.
