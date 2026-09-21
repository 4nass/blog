---
author: ["Anass"]
title: "SAST in CI/CD Pipelines: When the Security Gate Becomes an Attack Surface"
date: "2024-12-23"
description: "How SAST improves feedback in CI/CD, why the pipeline itself can be attacked, and how to build safer scans, gates, runners and reports."
tags: ["SAST", "DevSecOps", "CI/CD", "Software Supply Chain", "Pipeline Security"]
categories: ["Security"]
topics: ["Security"]
content_type: "security-analysis"
ShowToc: true
---

Static Application Security Testing is often introduced as a simple control: a developer opens a pull request, the scanner analyses the source code, and the pipeline blocks dangerous changes. That model is useful, but incomplete.

A SAST job runs inside a build system that can read source code, produce security evidence and sometimes access tokens, artifacts or deployment credentials. If an attacker can change the workflow, influence the scanner input, poison the baseline or execute code on the runner, the security gate itself becomes an attack surface.

The right goal is not to make every pipeline fail. It is to make the security decision trustworthy: the code being scanned is the code under review, the scanner and rules are controlled, the runner has minimal authority, findings cannot be silently suppressed, and a failed control cannot be converted into a successful release by accident.

<!--more-->

## The short version

- SAST analyses source code without running the application and can provide fast feedback during development.
- A SAST result is evidence, not a complete security verdict. It does not replace review, dependency analysis, secret scanning, testing or runtime controls.
- The pipeline is privileged infrastructure. Workflow changes, third-party actions, runners, caches, artifacts and tokens must be protected.
- Never execute untrusted pull-request code in a privileged workflow that can access repository secrets or write to protected branches.
- Baselines and suppressions must be reviewed like code, scoped narrowly and given an owner or expiry.
- Block on new, actionable risk with an explicit policy; do not hide an unmanageable legacy backlog behind `|| true`.
- Generate evidence that can be traced to a commit, scanner version, ruleset and workflow revision.

> **Lab safety:** the demonstrations below are local policy models and defensive workflow examples. Do not run untrusted code with production credentials, on a persistent self-hosted runner or in a workflow that has write access to a protected repository.

## At a glance

| Field | Details |
| --- | --- |
| Control | Static Application Security Testing |
| Analysis target | Human-readable source code and selected configuration |
| Main benefit | Early, repeatable feedback on security patterns |
| Main limitation | False positives, false negatives and incomplete runtime context |
| Pipeline asset | Source, findings, artifacts, tokens and runner environment |
| Common bypass | Workflow change, fail-open error handling or unreviewed suppression |
| High-risk trigger | Privileged workflow checking out untrusted pull-request code |
| Safer baseline | Read-only permissions, ephemeral runner, pinned tooling and no secrets |
| Release decision | New, verified findings evaluated against an explicit policy |

## What SAST can and cannot tell you

SAST rules look for patterns in source code and, depending on the engine, reason about data flow, control flow, types, frameworks and configuration. Useful detections can include:

- untrusted input reaching a dangerous interpreter or sink;
- missing authorization checks around a sensitive operation;
- insecure cryptographic or TLS configuration;
- path traversal, injection and unsafe deserialization patterns;
- hard-coded secrets or credentials;
- insecure use of a framework API;
- security-relevant configuration mistakes in code or infrastructure files.

The result depends on the language, framework, build model, ruleset and context available to the analyzer. A clean scan means “no matching issue was reported under this configuration”, not “the application is secure”.

SAST also has blind spots. It may not observe:

- vulnerabilities in third-party dependencies, which require SCA and vulnerability intelligence;
- runtime behaviour that depends on deployment configuration or external services;
- authorization errors that require business context;
- secrets injected only at deployment time;
- a compromised build tool, runner or dependency;
- a workflow that never runs the scan on the branch or file that changed.

Use SAST as one signal in a broader software-security process. The [NIST SSDF](https://csrc.nist.gov/projects/ssdf) places automated code analysis alongside controlled build processes, review, provenance and vulnerability response rather than treating one scanner as the entire program.

## The real SAST attack surface

A pipeline scan has more moving parts than the command that invokes the analyzer:

1. **Source control:** branch protections, pull-request permissions and workflow files.
2. **Trigger:** `push`, `pull_request`, `pull_request_target`, scheduled or release events.
3. **Checkout:** which commit and repository are placed on the runner.
4. **Runner:** operating-system state, network access, caches and persistence.
5. **Toolchain:** scanner binary, rules, plugins, package manager and action dependencies.
6. **Configuration:** ignore files, baselines, severity thresholds and custom rules.
7. **Output:** SARIF, logs, annotations, artifacts and pull-request comments.
8. **Gate:** the code that converts findings or scanner errors into pass/fail.
9. **Downstream jobs:** packaging, signing, publishing and deployment.

An attacker does not need to defeat the SAST algorithm if they can alter one of the surrounding decisions. The security property to protect is the chain from source revision to release decision.

## The attack scenario: poisoned SAST in a pull request

Imagine a repository where a contributor can open a pull request and the SAST workflow runs automatically.

### 1. The change enters through an apparently normal pull request

The attacker adds a vulnerable code path, a modified workflow file or a change to the scanner configuration. The pull request looks like an ordinary feature or dependency update.

### 2. The workflow runs with more authority than the change needs

The job uses a privileged trigger, checks out the pull-request branch and exposes repository secrets or a write-capable token. A build step, package script or custom scanner hook now has an opportunity to execute attacker-controlled code.

GitHub documents this class of risk for workflows that combine privileged contexts such as `pull_request_target` or `workflow_run` with an untrusted checkout. The problem is not the name of the SAST product; it is the relationship between untrusted content and privileged execution.

### 3. The attacker weakens the control

Several outcomes are possible without attacking the analyzer itself:

- the workflow is changed to run the scan with `|| true`;
- a severity threshold is raised so that the new issue is not blocking;
- a broad path exclusion hides the vulnerable file;
- the baseline is regenerated after the finding appears;
- the scan runs only on the default branch and not on the pull request;
- a scanner error is treated as a successful job;
- the report is uploaded but never consumed by the release gate.

### 4. The pipeline produces reassuring evidence

The job is green, a comment says “no new findings”, and the artifact contains a report. The evidence is still untrustworthy if the workflow, ruleset or commit under analysis was controlled by the same untrusted change.

### 5. The release path accepts the result

If the approval process trusts the green check without verifying the workflow revision, branch protection and report provenance, vulnerable code can progress to packaging or deployment.

This scenario is a pipeline-integrity failure. It is not a claim that every SAST scanner is exploitable, and it does not require a false negative in the analyzer. The surrounding automation can invalidate a correct scanner.

## A safe local policy-gate demonstration

The following command is deliberately named `sast-new-findings-gate`. It reads a small, normalized report format and blocks only on new open findings with `high` or `critical` severity. It does not scan code, execute project files or contact a remote service.

Save the example report as `sast-report.json`:

```json
{
  "commit": "local-lab",
  "scanner": "example-sast",
  "findings": [
    {"id": "SQL-001", "severity": "high", "status": "open", "baseline": false},
    {"id": "STYLE-004", "severity": "low", "status": "open", "baseline": false},
    {"id": "LEGACY-002", "severity": "critical", "status": "open", "baseline": true}
  ]
}
```

Run the gate:

```bash
#!/usr/bin/env bash
set -euo pipefail

COMMAND_NAME="sast-new-findings-gate"
REPORT="${1:-sast-report.json}"

echo "== ${COMMAND_NAME} =="
echo "report=${REPORT}"

if ! command -v jq >/dev/null 2>&1; then
  echo "error: jq is required"
  exit 2
fi

if [[ ! -r "${REPORT}" ]]; then
  echo "error: report is not readable"
  exit 2
fi

blocking_count="$(jq '[.findings[]
  | select(.status == "open")
  | select(.baseline == false)
  | select(.severity == "high" or .severity == "critical")]
  | length' "${REPORT}")"

if [[ "${blocking_count}" -gt 0 ]]; then
  echo "status=blocked new_high_or_critical=${blocking_count}"
  exit 1
fi

echo "status=passed new_high_or_critical=0"
```

This model illustrates two important choices:

- historical findings do not have to block every pull request once they are tracked in a baseline;
- a baseline does not make a new finding disappear, because the gate distinguishes `baseline: true` from `baseline: false`.

In a real implementation, normalize the scanner’s native format carefully and keep the report schema versioned. Never assume that every tool uses the same severity names or baseline semantics.

The dangerous anti-pattern is much shorter:

```bash
# A scanner failure and a clean result become indistinguishable.
sast-cli scan . || true
```

If availability is a concern, make the exception explicit and observable. A scanner outage should normally produce a failed or “security check unavailable” state, not a green release.

## A safer CI workflow shape

The exact syntax depends on the CI provider, but the security properties should be consistent:

```yaml
name: sast

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  scan:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read

    steps:
      # Pin actions to reviewed full commit SHAs in the real repository.
      - name: Checkout source
        uses: actions/checkout@<reviewed-full-commit-sha>
        with:
          fetch-depth: 1

      - name: Run SAST
        run: |
          set -euo pipefail
          sast-cli scan . --format sarif --output sast.sarif

      - name: Upload report for review
        if: always()
        uses: actions/upload-artifact@<reviewed-full-commit-sha>
        with:
          name: sast-report
          path: sast.sarif
          retention-days: 7
```

The placeholder action references are intentional. Replace them with reviewed, immutable commit SHAs rather than floating tags after verifying the action source and update process.

This job should not need repository secrets, a write-capable `GITHUB_TOKEN`, deployment credentials or access to a persistent production network. A scan that only reads source should have read-only permissions.

If the analyzer requires compilation or dependency installation, isolate that work further. Treat package lifecycle scripts, build hooks and generated code as untrusted when the source comes from a fork or external contribution. Do not use a privileged workflow to check out untrusted code merely because the scan needs an artifact.

## GitHub trigger boundaries

On GitHub, `pull_request` and `pull_request_target` have materially different security properties. A common dangerous pattern is:

```yaml
on: pull_request_target

steps:
  - uses: actions/checkout@...
    with:
      ref: ${{ github.event.pull_request.head.sha }}
  - run: npm install
  - run: npm test
```

This combines a privileged workflow context with code selected by the pull request. The package install and test commands can execute code from that change. The same concern applies to a SAST job if it performs a build or loads project-controlled plugins.

Safer patterns include:

- run source-only analysis from `pull_request` with read-only permissions;
- keep secrets out of jobs that process untrusted code;
- use a separate trusted job to publish a result, with strict validation of the artifact and source revision;
- never let untrusted input control a shell fragment, action reference or artifact extraction path;
- restrict who can approve workflow changes and who can modify branch-protection rules.

GitHub’s [secure-use guidance](https://docs.github.com/en/actions/reference/security/secure-use) and [workflow execution protections](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/actions-policies/workflow-execution-protections) describe these risks in detail.

## Protecting baselines and suppressions

Baselining is useful for introducing SAST into a legacy codebase. It becomes dangerous when it is treated as an invisible exception list.

For every suppression or baseline entry, record:

- the finding identifier and precise location;
- the reason it is accepted or considered a false positive;
- the risk owner and review date;
- the compensating control, if any;
- the planned remediation or expiry condition.

Protect the file with normal code review and ownership rules. Do not allow a pull request to both introduce a finding and regenerate the baseline that hides it. A trusted job can compare the proposed report with the baseline from the protected default branch, then require an explicit approval for changes to the baseline.

The most robust policy is usually “no new verified high-risk findings” rather than “the entire repository must be clean immediately”. That policy preserves delivery while keeping the security debt visible.

## Runner and toolchain security

The runner is part of the trusted computing base of the pipeline. Harden it according to the trust level of the job:

- prefer ephemeral runners for untrusted pull requests;
- do not reuse workspaces or caches between unrelated contributors;
- restrict outbound network access where the scan does not need it;
- avoid storing credentials in environment variables exposed to child processes;
- remove temporary reports and source copies after the job;
- pin scanner versions, rulesets and action dependencies;
- verify downloaded tools and rules before execution;
- separate analysis runners from signing, publishing and deployment runners;
- monitor changes to runner images and privileged workflow definitions.

The [SLSA build-environment guidance](https://slsa.dev/spec/draft/build-env-track-basics) treats the build platform as privileged because it can access sensitive operations such as artifacts and provenance signing. A SAST job may not sign releases itself, but it still runs inside that same supply-chain control plane.

## Handling reports safely

SAST reports can contain source fragments, file paths, usernames, URLs and occasionally secrets discovered in code. Treat them as sensitive build artifacts:

- limit who can read raw reports;
- avoid publishing full reports to public pull requests;
- redact secrets before sending annotations to external systems;
- set short retention for temporary artifacts;
- distinguish a developer-friendly summary from the authoritative machine-readable report;
- bind the report to the exact commit, workflow run, scanner version and ruleset digest;
- prevent artifact names and extraction paths from being controlled by untrusted input.

Do not let a pull-request comment become the source of truth. The authoritative evidence should remain in a protected CI store or security platform with an auditable relationship to the run.

## A practical rollout strategy

### Phase 1: observe

Run SAST on pull requests and the default branch without blocking. Measure scan duration, language coverage, false-positive rate and the number of findings that developers can actually act on.

### Phase 2: baseline carefully

Review existing findings, document accepted risk and create a protected baseline. Keep the initial report available so the security debt is not forgotten.

### Phase 3: block new actionable risk

Fail the pull request on new, verified critical or high findings. Keep the rule narrow enough that developers trust the signal, but do not convert tool errors into success.

### Phase 4: separate release assurance

Run a full scan on the protected default branch and before release. The release scan should use a trusted workflow definition and a controlled toolchain rather than trusting the workflow supplied by the feature branch.

### Phase 5: improve coverage

Add SCA, secret scanning, IaC analysis, container scanning, DAST and manual review where appropriate. Feed production incidents and accepted exceptions back into custom rules and secure coding guidance.

## Metrics that matter

Counting findings alone encourages teams to optimize the wrong thing. Track:

- median and 95th-percentile scan duration;
- findings introduced, fixed and reopened per period;
- time to triage and time to remediate by severity;
- false-positive rate by rule;
- percentage of repositories scanned on every relevant change;
- number of suppressed findings with an owner and expiry;
- scanner and ruleset versions used by each protected branch;
- rate of scan failures or unavailable security checks;
- percentage of release artifacts linked to a successful trusted scan.

The last two metrics are especially important. A team that reports a high scan-coverage percentage but silently ignores scanner failures may have less assurance than a team with a smaller, honest coverage figure.

## What this means for developers and security teams

Developers need findings that are precise, explainable and close to the change that introduced them. Security teams need a gate that cannot be weakened by the same untrusted change it is evaluating. Platform teams need runners, actions, tokens and artifacts governed as production infrastructure.

Those needs are compatible when responsibilities are separated:

- the feature workflow provides fast, read-only feedback;
- protected automation produces authoritative release evidence;
- humans approve risk exceptions and baseline changes;
- the toolchain is versioned and traceable;
- the policy distinguishes legacy debt from new risk without hiding either one.

## Conclusion

SAST is valuable because it moves useful security feedback closer to the code. Its limits become visible when the pipeline is treated as a neutral transport instead of a privileged system.

The scanner is only one part of the control. Protect the trigger, checkout, runner, toolchain, rules, baseline, report and gate. Keep untrusted code away from secrets and write permissions. Make scanner failures visible. Tie every decision to a commit and a trusted workflow revision.

The strongest SAST integration is not the one with the most rules or the most red builds. It is the one whose results developers can act on and whose security decision remains trustworthy when the change under review is hostile.

## Sources

1. [OWASP Source Code Analysis Tools](https://owasp.org/www-community/Source_Code_Analysis_Tools), overview of SAST tools and approaches.
2. [NIST Secure Software Development Framework](https://csrc.nist.gov/projects/ssdf), practices for secure development and automated analysis.
3. [GitHub — Secure use of Actions](https://docs.github.com/en/actions/reference/security/secure-use), untrusted input, workflow and secret risks.
4. [GitHub — Workflow execution protections](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/actions-policies/workflow-execution-protections), controls for workflow triggers and poisoned pipeline execution.
5. [SLSA — Build Environment track](https://slsa.dev/spec/draft/build-env-track-basics), trust and privilege in build platforms.
