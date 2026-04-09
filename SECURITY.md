# Security Policy

## Reporting a Vulnerability

**DO NOT** open a public issue for security vulnerabilities.

Report via [GitHub Private Vulnerability Reporting](../../security/advisories/new)
or email franchb@users.noreply.github.com.

## Scope

### In Scope

- **Prompt injection**: Content that could cause Claude Code to execute
  unintended actions, exfiltrate data, or bypass user consent
- **Data exfiltration**: Instructions that could send data to unauthorized endpoints
- **Privilege escalation**: Content causing actions beyond the skill's declared
  `allowed-tools` scope (Read, Grep, Glob, LSP)
- **Hidden/obfuscated instructions**: Unicode tricks, zero-width characters,
  or steganographic techniques hiding malicious instructions
- **Supply chain compromise**: Unauthorized modifications to skill files,
  plugin manifest, or CI workflows

### Out of Scope

- General Claude Code platform vulnerabilities
  (report to [Anthropic's security team](https://www.anthropic.com/security))
- Bugs in fp-go/v2 library itself
  (report to [IBM/fp-go](https://github.com/IBM/fp-go))

## Disclosure Timeline

- Acknowledgment within **48 hours**
- Initial assessment within **5 business days**
- Fix within **30 days** (critical) / **90 days** (lower severity)
- Coordinated 90-day disclosure

## Verification for Users

Verify release attestation:

```bash
gh attestation verify <artifact> -R franchb/fp-go-skill
```

Verify SLSA provenance:

```bash
slsa-verifier verify-artifact <artifact> \
  --provenance-path multiple.intoto.jsonl \
  --source-uri github.com/franchb/fp-go-skill \
  --source-tag <tag>
```
