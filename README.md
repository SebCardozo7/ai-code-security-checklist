# Shipping AI-Generated Code: A Pre-Production Security Checklist

A review checklist for code written by LLMs and coding agents, aimed at the moment before it reaches production.

**[Use the interactive version →](https://loop.github.io/ai-code-security-checklist/)**

---

## Why a separate checklist

AI-generated code does not fail the way junior-developer code fails. It fails in a specific and predictable set of ways, and those are what this list targets.

It is fluent. It compiles, it reads well, it uses the right vocabulary, and it passes the tests that were written alongside it — often because those tests were generated from the same flawed understanding. Ordinary code review leans heavily on "does this look like someone who knew what they were doing wrote it," and that signal no longer carries information.

It is plausible rather than verified. Models produce package names, API signatures, and configuration flags that look correct and sometimes do not exist. Attackers have learned to register the package names that models invent most often.

It optimizes for the happy path. Given a feature request, a model writes the feature. Authorization checks, rate limits, timeouts, and failure branches are frequently absent, because nothing in the prompt asked for them.

It has no memory of your system. Constraints that live in a threat model, a runbook, or a senior engineer's head are not in the context window.

The checklist below assumes a human reviewer who understands the system. It does not replace one.

---

## How to use this

Run the automated checks first (section 12). They clear roughly half of what follows and take minutes. Spend human attention on what remains.

Severity is marked to help triage, not to give permission to skip:

- **Blocking** — do not merge until resolved.
- **High** — resolve before the release this ships in.
- **Track** — record it; fix it on a schedule.

---

## 1. Dependencies and supply chain

**Blocking** — Every imported package exists and is the package intended. Search the registry by exact name. Models hallucinate package names, and squatters register the popular hallucinations.

**Blocking** — No new dependency was added for something the codebase already does. Check for an existing internal utility first.

**High** — Each new dependency has a plausible history: recent releases, an identifiable maintainer, a download volume consistent with its claimed purpose.

**High** — Versions are pinned and the lockfile is committed.

**High** — A vulnerability scan has run against the new dependency tree (`osv-scanner`, `npm audit`, `pip-audit`, `cargo audit`).

**Track** — The dependency count is justified. Agents add libraries freely; the aggregate cost lands on whoever maintains this later.

## 2. Secrets and configuration

**Blocking** — No credentials, API keys, tokens, or connection strings in source, including in comments, example files, fixtures, and test data.

**Blocking** — If a secret was ever committed, it is rotated. Removing it from the working tree does not remove it from history.

**High** — Secrets are read from environment variables or a secret manager, not from a config file in the repo.

**High** — A secret scanner has run over the diff and the branch history (`gitleaks`, `trufflehog`).

**High** — Debug flags, verbose logging, and permissive CORS settings introduced during development are off.

## 3. Input handling and injection

**Blocking** — Database access uses parameterized queries or an ORM's safe interface. No SQL assembled by string concatenation or interpolation.

**Blocking** — Shell and subprocess calls pass arguments as arrays, never as an interpolated command string. `shell=True` and equivalents are absent or justified in writing.

**Blocking** — Any file path derived from user input is resolved and confirmed to stay inside the intended directory.

**High** — Untrusted data is not deserialized with formats that can instantiate arbitrary types (`pickle`, unsafe YAML loaders, Java native serialization).

**High** — Template output is escaped by default, and any escaping that was disabled has a stated reason.

**High** — Outbound requests built from user-supplied URLs validate the destination against an allowlist. Check this specifically for webhooks, URL previews, image fetchers, and imports.

**High** — Input validation happens server-side. Client-side validation the model added is a usability feature, not a control.

## 4. Authentication and authorization

**Blocking** — Every new route, handler, and RPC method has an explicit authorization check. This is the single most common gap in generated code.

**Blocking** — Object-level authorization is enforced: the handler confirms the requested resource belongs to the caller, not merely that the caller is logged in. Look at every endpoint that accepts an ID.

**High** — Authentication uses the framework's or provider's primitives. Any hand-rolled token, session, or signature logic is rewritten.

**High** — Session cookies are `Secure`, `HttpOnly`, and `SameSite`-scoped. Tokens have expiry and a revocation path.

**High** — Privileged operations are not selected by a client-supplied role, flag, or parameter.

**Track** — Authorization logic is centralized rather than repeated per handler.

## 5. Cryptography

**Blocking** — No custom cryptographic construction. If the diff contains a hand-built encryption, signing, or key-derivation routine, replace it with a vetted library.

**Blocking** — Passwords are hashed with Argon2, scrypt, or bcrypt. Not SHA-256, not SHA-256 with a salt, not anything faster.

**High** — Randomness used for tokens, session IDs, nonces, or keys comes from a cryptographically secure source (`crypto.randomBytes`, `secrets`, `os.urandom`), never from a general-purpose PRNG.

**High** — Modern algorithms and modes only. No ECB, no MD5 or SHA-1 for security purposes, no hardcoded IVs.

**High** — TLS certificate verification is enabled. Any `verify=False`, `rejectUnauthorized: false`, or `InsecureSkipVerify` is removed.

## 6. Error handling and failure behavior

**Blocking** — Failures fail closed. When an authorization check, a token validation, or an external policy call errors, the result is denial, not a fallthrough to allow.

**High** — Errors returned to clients carry no stack traces, query fragments, file paths, or internal hostnames.

**High** — No empty or blanket catch blocks. Generated code swallows exceptions to make a function "work"; each one hides a failure mode.

**High** — Logs contain no secrets, tokens, full request bodies, or personal data.

**Track** — New failure paths emit something observable. A silent degradation is worse than an error.

## 7. Concurrency, resources, and limits

**High** — Shared mutable state accessed from concurrent paths is protected. Check-then-act sequences on balances, quotas, inventory, and idempotency keys are the usual sites.

**High** — Every outbound network call has a timeout. Defaults in most HTTP clients are unbounded.

**High** — Connections, file handles, and spawned workers are released on every path, including error paths.

**High** — Expensive endpoints and authentication endpoints are rate limited.

**Track** — Loops, recursion, and pagination over user-controlled input have a bound.

## 8. Data handling

**High** — API responses return only the fields the client needs. Generated serializers commonly expose the entire model, including internal flags and adjacent user data.

**High** — Personal data introduced by this change is accounted for under whatever regime applies to you, including retention and deletion.

**High** — Deletion deletes. Soft-delete that leaves data reachable through another query path is not deletion.

**Blocking** — No data leaves for an endpoint, analytics service, or SDK that the model introduced and nobody asked for.

## 9. If the system itself calls an LLM

**Blocking** — Model output is treated as untrusted input. It does not reach `eval`, `exec`, a shell, a SQL string, or a template without the same handling any user input would get.

**Blocking** — Side-effectful tool calls are gated by policy or human approval rather than decided by the model alone. Content retrieved from documents, web pages, and emails can carry instructions.

**High** — Prompts assembled from user input keep a clear boundary between instruction and data, and the system does not rely on that boundary alone for security.

**High** — Token, cost, and request-rate limits exist per user and in aggregate.

**Track** — Prompts and responses stored for debugging fall under the same data-handling rules as anything else.

## 10. Tests

**High** — Tests assert behavior, not that a function returns without throwing.

**High** — Negative cases exist: unauthorized caller, malformed input, missing resource, downstream failure.

**High** — Tests were not written to match the implementation's bug. Read the assertion against the requirement, not against the code.

**Track** — The failure branches the model wrote are exercised by something.

## 11. Provenance and licensing

**High** — Substantial verbatim blocks are checked against public sources. Models reproduce training data, and some of it is copyleft.

**Track** — Your organization's position on AI-generated code is recorded where reviewers can see it, and the PR states what was generated.

## 12. Automate these

Run before human review. Everything here is mechanical and catching it by eye is a waste of an engineer.

| Check | Tools |
|---|---|
| Secret scanning | gitleaks, trufflehog |
| Dependency vulnerabilities | osv-scanner, Dependabot, npm audit, pip-audit |
| Static analysis | Semgrep, CodeQL |
| Container and IaC scanning | Trivy, Checkov, tfsec |
| License compliance | ScanCode, FOSSA |
| Type and lint gates | language-native, failing the build |

Two things worth wiring specifically for generated code: a registry-existence check on new dependencies, and a rule that flags new HTTP handlers lacking an authorization decorator or middleware.

## 13. Before merge

**Blocking** — A named human who understands the system owns this change and can explain each decision in it.

**Blocking** — The diff is small enough to be reviewed. A 2,000-line agent-generated PR is not reviewed by anyone; split it.

**High** — Automated gates in section 12 passed, and failures were fixed rather than suppressed.

**High** — Anything deferred is written down as an issue, with a severity and an owner.

---

## Contributing

Corrections and additions are welcome, particularly failure modes observed in real review. Open an issue or a PR. Items should be specific enough to check and general enough to apply beyond one language.

## Further reading

- [NIST SP 800-218, Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [What the NIST SSDF means for product teams](https://loopstudio.dev/) — LoopStudio
- [Securing AI-built software before it goes to production](https://loopstudio.dev/) — LoopStudio

## License

MIT. Use it, fork it, put it in your internal wiki.
