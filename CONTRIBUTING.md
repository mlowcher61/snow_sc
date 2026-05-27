# Contributing to snow_sc

Thanks for your interest in improving this ServiceNow → Event-Driven Ansible solution. This guide
covers how to set up, the conventions this repo follows, and how to submit changes.

## Project conventions

- **Target Ansible Automation Platform 2.5**, not bare `ansible-core`. Assume execution
  environments, the Controller, and EDA.
- Prefer **certified/validated collections** (`servicenow.itsm`, `ansible.eda`,
  `ansible.platform`, `infra.aap_configuration`) over community equivalents.
- **No secrets in git.** Credentials are supplied at runtime via AAP custom credential types (see
  `aap_config/files/controller_credential_types.yml`). Never commit tokens, passwords, or vault
  files. The only token strings in this repo are obvious placeholders (e.g. `ABC123`).
- **Config-as-code** lives in `aap_config/` so a new user can stand up the AAP objects on their
  own platform.

## Setup

```bash
git clone https://github.com/mlowcher61/snow_sc.git
cd snow_sc
ansible-galaxy collection install -r collections/requirements.yml
pip install ansible-lint yamllint        # linters used in checks below
```

## Before you open a PR

Run the linters from the repo root — both must be clean:

```bash
yamllint .
ansible-lint
```

Sanity-check playbook syntax (requires the collections installed above):

```bash
ansible-playbook playbooks/configure_servicenow.yml --syntax-check
ansible-playbook playbooks/configure_eda.yml --syntax-check
```

If your change touches a role's behavior, confirm **idempotency** against a test instance: a second
run of the playbook should report `changed=0`. Use throwaway test credentials passed with `-e` (or
a local `-e @vars.yml` file that is git-ignored) — never commit them.

## Making changes

1. Branch from `main`: `git checkout -b feat/<short-description>`.
2. Keep commits small and focused, one logical change each. Use clear messages
   (`feat(snow): ...`, `feat(eda): ...`, `fix: ...`, `docs: ...`).
3. Put tunable values in the relevant role's `defaults/main.yml` — don't hardcode names, filters,
   or sys_ids in task files.
4. Update `README.md` and any doc in `docs/` if your change affects how the solution is deployed
   or used.
5. Open a PR against `main` describing what changed and how you verified it.

## ServiceNow / AAP specifics to keep in mind

- ServiceNow platform artifacts (Outbound REST Message, Business Rule, Catalog Item) are managed
  through the generic `servicenow.itsm.api` module against the Table API. Table and field names can
  vary by ServiceNow version — verify with `servicenow.itsm.api_info` before relying on a new one.
- The Flow Designer flow is intentionally **not** automated; document any changes to it in
  `docs/flow_designer_manual_steps.md`.
- EDA event streams and `ansible.platform.*` modules vary by collection version. If a module isn't
  available, prefer a documented fallback over a fragile workaround (see `eda_create_event_stream`
  in `roles/eda_snow_config/defaults/main.yml`).

## Reporting issues

Open a GitHub issue with: what you expected, what happened, your AAP and collection versions, and
the relevant playbook output (with secrets redacted).

## License

By contributing, you agree that your contributions are licensed under the [MIT License](LICENSE).
