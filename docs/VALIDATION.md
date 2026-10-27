# Publication validation

Publication checks are limited to this documentation/source bundle. They do not add hardware validation to the 2026-10-25 run.

The checklist for publication:
- Verify remote source, retained trial report and persistent installation records.
- Confirm the published hook and CUDA templates differ from the tested sources only by PCI-address substitution.
- Parse all JSON/JSONL; reconcile completed-case counts, failed cases, hashes and explicit null performance claim.
- Syntax-check the expanded shell hook; compile the expanded CUDA template and its small-smoke variant on the original Linux host without executing a GPU workload.
- Check local Markdown links and reject private network addresses, usernames, host/device UUIDs, credentials, build products and firmware in the publication allowlist.
- Run a redacted credential scan on this repository only.
- Read back the GitHub tree and document contents after publication.

The final check results are recorded in evidence/publication-validation.json. That file reports publication checks, not new GPU tests.

No firmware assets, .ko, initramfs or captured full host logs are included. Compilation checks do not establish firmware safety, cross-machine compatibility or long-term stability.
