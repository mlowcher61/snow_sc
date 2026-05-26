---
name: Bug report
about: Something in the ServiceNow → EDA solution isn't working as expected
title: "[Bug] "
labels: bug
assignees: ''
---

## What happened

A clear description of the problem.

## What you expected

What you expected to happen instead.

## Steps to reproduce

1. Ran `...` (playbook / job template)
2. With these inputs (redact secrets) `...`
3. Saw `...`

## Component

- [ ] `roles/snow_catalog_config` (ServiceNow side)
- [ ] `roles/eda_snow_config` (EDA side)
- [ ] `rulebooks/servicenow_catalog.yml`
- [ ] `configure_aap/` (config-as-code)
- [ ] Docs / README

## Environment

- AAP version (e.g. 2.5):
- Collection versions (`ansible-galaxy collection list servicenow.itsm ansible.eda ansible.platform`):
- ServiceNow version / release:
- ansible-core version:

## Playbook output

```text
Paste the relevant output here. REDACT instance URLs, tokens, usernames, and passwords.
```

## Additional context

Anything else that helps — screenshots of the ServiceNow record, the event stream config, etc.
