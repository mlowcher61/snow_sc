# snow_sc — ServiceNow Service Catalog → Event-Driven Ansible

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Ansible Automation Platform](https://img.shields.io/badge/Ansible%20Automation%20Platform-2.5-EE0000?logo=ansible&logoColor=white)](https://www.redhat.com/en/technologies/management/ansible)

Trigger Ansible Automation Platform automation from a ServiceNow Service Catalog request. When an
approved Requested Item reaches **Work in Progress**, a ServiceNow Business Rule posts the event to
an EDA event stream, and a rulebook runs a job/workflow template.

## How it works

```
[Catalog item] --order--> [Flow: approval] --> sc_req_item = Work in Progress
                                                   |
                              [Business Rule] --> Outbound REST Message --POST-->
                                                   [EDA Event Stream] --> [Rulebook]
                                                   --> run_job_template / run_workflow_template (by name)
```

## What this repo automates

| Piece | Automated by | Notes |
|-------|--------------|-------|
| Outbound REST Message + headers | `roles/snow_catalog_config` | via `servicenow.itsm.api` |
| Business Rule (sc_req_item) | `roles/snow_catalog_config` | templated RESTMessageV2 script |
| Catalog Item | `roles/snow_catalog_config` | created + published |
| Flow Designer flow | **manual** | see `docs/flow_designer_manual_steps.md` |
| EDA credential type / credential / event stream | `roles/eda_snow_config` | via `ansible.eda` |
| EDA project + rulebook activation | `roles/eda_snow_config` | references this repo |
| AAP credential types, project, job templates, surveys | `configure_aap/` | via `ansible.platform` |

## Prerequisites
- AAP 2.5 (Controller + EDA), ServiceNow instance with admin, `ansible-core` ≥ 2.15.
- `ansible-galaxy collection install -r collections/requirements.yml`

## Deploy (UI / config-as-code)
1. Bootstrap AAP: see `configure_aap/README.md`.
2. Run **Configure EDA** first — copy the event stream endpoint it prints.
3. Run **Configure ServiceNow Catalog**, providing that endpoint in the survey.
4. Build and link the Flow Designer flow: `docs/flow_designer_manual_steps.md`.
5. Order the catalog item, approve it, and confirm the rulebook activation runs your template.

## Deploy (CLI, for testing)
```bash
ansible-playbook playbooks/configure_eda.yml \
  -e eda_host=https://<aap> -e eda_username=admin -e eda_password=*** -e eda_token=ABC123
# copy the printed endpoint, then:
ansible-playbook playbooks/configure_servicenow.yml \
  -e snow_instance=https://<inst>.service-now.com -e snow_username=admin -e snow_password=*** \
  -e snow_eda_endpoint=<endpoint> -e snow_eda_token=ABC123 \
  -e snow_catalog_item_category_sys_id=<sys_id>
```

## Secrets
Secrets are supplied at launch via AAP **custom credentials** (ServiceNow Connection + EDA Event
Stream Token) — never vaulted in git. The same token must match on both sides.

## Configuration
All names, filters, and payload fields are in each role's `defaults/main.yml`. Override via survey
or `-e`.
