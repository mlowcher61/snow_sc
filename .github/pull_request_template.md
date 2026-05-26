## Summary

What does this PR change, and why?

## Component(s)

- [ ] `roles/snow_catalog_config` (ServiceNow side)
- [ ] `roles/eda_snow_config` (EDA side)
- [ ] `rulebooks/servicenow_catalog.yml`
- [ ] `configure_aap/` (config-as-code)
- [ ] Docs / README

## How verified

- [ ] `yamllint .` passes
- [ ] `ansible-lint` passes
- [ ] `ansible-playbook <playbook> --syntax-check` passes
- [ ] Idempotency confirmed (second run reports `changed=0`) against a test instance
- [ ] Docs updated (`README.md` / `docs/`) if behavior or deployment changed

Describe what you ran and the result (redact secrets):

```text
```

## Notes for reviewers

Anything version-specific (AAP / ServiceNow / collection versions), assumptions, or follow-ups.

## Checklist

- [ ] No secrets committed (tokens, passwords, vault files)
- [ ] Tunable values live in the relevant role's `defaults/main.yml`, not hardcoded in tasks
- [ ] Commits are focused with clear messages
