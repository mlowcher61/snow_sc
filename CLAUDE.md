# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

A working **ServiceNow Service Catalog → Event-Driven Ansible** integration for AAP 2.5. An approved
Requested Item fires a ServiceNow Business Rule that posts to an EDA event stream, and a rulebook
runs a job/workflow template. See `README.md` for the end-to-end walkthrough.

## Architecture

- **`roles/snow_catalog_config`** — uses `servicenow.itsm.api` to create the Outbound REST Message,
  Business Rule, and Catalog Item idempotently (query-then-create).
- **`roles/eda_snow_config`** — uses `ansible.eda` to create the EDA credential type, credential,
  event stream, project, and rulebook activation.
- **`rulebooks/servicenow_catalog.yml`** — webhook source that runs a template by name on a matching
  payload.
- **`playbooks/configure_servicenow.yml` / `configure_eda.yml`** — thin wrappers that run the two
  roles; these are the playbooks the job templates launch.
- **Flow Designer flow** — intentionally NOT automated (`servicenow.itsm` can't build it via the
  Table API); documented as manual steps in `docs/flow_designer_manual_steps.md`.

## Config-as-code — `aap_config/`

Stands up every AAP object on a new platform via the certified **`infra.aap_configuration`**
collection's `dispatch` role: you declare *what* should exist in `files/*.yml` data structures, and
`dispatch` reconciles AAP to match (idempotent). Modeled on the layout at
`github.com/ericcames/dc1.azure/tree/main/aap_config`.

```
aap_config/
├── load.yml          # entry point: applies files/ via infra.aap_configuration.dispatch, then validate.yml
├── validate.yml      # post-load API check — fails if any expected object is missing
├── requirements.yml  # infra.aap_configuration, ansible.platform, ansible.eda
├── inventory/aap.yml # localhost — the CaC run talks to AAP over the API
├── group_vars/all.yml# connection (AAP_HOSTNAME/AAP_TOKEN), object names, secrets — all env-var lookups
└── files/            # declarative data structures consumed by dispatch:
    ├── gateway_organizations.yml        # aap_organizations
    ├── controller_credential_types.yml  # controller_credential_types (injectors use !unsafe)
    ├── controller_credentials.yml       # controller_credentials (filled from env vars)
    ├── controller_projects.yml          # controller_projects
    ├── controller_job_templates.yml     # controller_templates (note: NOT controller_job_templates)
    └── eda_projects.yml                 # eda_projects
```

Conventions when editing `aap_config/`:
- **Object names + repo URL/branch** live in `group_vars/all.yml`; the **shape** of each object lives
  in the matching `files/*.yml`. Edit the data file, re-run `load.yml`, dispatch reconciles.
- **No secrets in the data files** — every sensitive input is an env-var lookup in `group_vars/all.yml`
  and lands in an AAP custom credential, never in git.
- Credential-type **injectors use the `!unsafe` YAML tag** (e.g. `!unsafe '{{ password }}'`) so the
  template reference is written verbatim and resolved by the Controller at launch.
- Some dispatch list variables don't match their filename — job templates use `controller_templates`,
  workflows use `controller_workflows`.

## Build Conventions

These are the standing conventions for this repository. Apply them when scaffolding or extending the solution.

- **Target Ansible Automation Platform (AAP), not ansible-core**, unless explicitly told otherwise. Assume execution environments, controller, and the platform UI/API — not raw `ansible-playbook` on a workstation.
- **Use the `ansible.platform` collection** for platform/controller automation, in preference to `ansible.controller`.
- **Prefer certified and validated content** from Automation Hub over Galaxy community collections when an equivalent exists.
- **Credentials live in AAP, not in git.** Use custom credential types attached to job templates rather than `ansible-vault` files committed to the repo. Never commit secrets.
- **Ship config-as-code.** Every new repo includes config-as-code (e.g., `ansible.platform`-based controller configuration) so a new user can stand up the required AAP objects — projects, job templates, credentials, inventories — for their own platform.
- **README.md is mandatory and beginner-facing.** It must fully explain the solution end to end and be comprehensible to someone new to Ansible.

## Repository Owner

Repos are hosted at github.com/mlowcher61.
