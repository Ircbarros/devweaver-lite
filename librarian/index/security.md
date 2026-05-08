# security domain index

Security references: OWASP LLM Top 10, security toolchain (bandit, pip-audit, gitleaks).
Use this index when the task involves security compliance, vulnerability scanning, or auth.

| ISBN | File | Topics | Summary |
|---|---|---|---|
| sec-001 | references/security/rule_owasp_llm_top10.md | OWASP, LLM Top 10, prompt injection, compliance | OWASP LLM Top 10 compliance rules |
| sec-002 | references/security/rule_security_toolchain.md | bandit, pip-audit, gitleaks, semgrep, scanning | Security toolchain setup and usage |

> Usage note: load sec-001 during FETCH_RULES phase for any AI/LLM task.
> Load sec-002 during VALIDATE phase as part of the security scan step.
> Both files are frequently loaded together - they count as 2 of the 3 R-PD-03 slots.
