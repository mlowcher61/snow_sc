# Design: ServiceNow Service Catalog → Event-Driven Ansible Integration

**Date:** 2026-05-26
**Repository:** `snow_sc` (github.com/mlowcher61)
**Target platform:** Ansible Automation Platform 2.5 (gateway + Controller + EDA)

## Problem

A ServiceNow Service Catalog item, once ordered and approved, must trigger automation in
Ansible Automation Platform. The integration spans two platforms:

- **ServiceNow** fires an Outbound REST Message (from a Business Rule on the Requested Item
  table) to an EDA event stream endpoint when a request reaches the right state.
- **Event-Driven Ansible** receives the payload on an event stream, matches it in a rulebook,
  and runs a job template or workflow template.

This solution automates the configurable pieces on both sides as reusable Ansible roles, plus a
config-as-code layer so a new user can deploy it on their own AAP instance.

## Goals

- Automate the ServiceNow Outbound REST Message, Business Rule, and Catalog Item.
- Automate the EDA credential type, credential, event stream, project, and rulebook activation.
- Ship the rulebook that maps ServiceNow payloads to AAP templates (referenced by name).
- Provide config-as-code (`ansible.platform`) to create the AAP objects and surveys.
- Provide a beginner-facing README and documented manual steps for the Flow Designer flow.

## Non-Goals

- Automating the Flow Designer flow via the Table API (not practical; documented as manual steps).
- Shipping a ServiceNow Update Set XML (alternative considered and rejected — see below).
- Building the downstream provisioning job/workflow templates themselves (out of scope; the
  rulebook references them by name and the user supplies them).

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Flow Designer | Automate REST message + business rule + catalog item; **document** the flow as manual steps | Flow records span many interrelated `sys_hub_*` tables and are not reliably buildable via the Table API. Keeps automated artifacts transparent and reviewable. |
| EDA tooling | `ansible.eda` collection | Certified config-as-code path for EDA in AAP 2.5. |
| Secrets / auth | AAP custom credential types + survey | Per standing convention: secrets injected via custom credentials attached to job templates, never vaulted in git. |
| SNOW platform config | `servicenow.itsm.api` (generic Table API module) | The `servicenow.itsm` collection has no dedicated modules for REST messages, business rules, or catalog items; the `api` module does idempotent CRUD against any table. |

## Architecture

```
ServiceNow side                          Ansible side (AAP 2.5)
───────────────                          ──────────────────────
[Catalog item] ──order──> [Flow] ──> sc_req_item update
                                          │
                          [Business Rule on sc_req_item]
                          fires Outbound REST Message ──POST──▶ [EDA Event Stream endpoint]
                                                                      │
                                                                [Rulebook activation]
                                                                match payload → run_job_template /
                                                                                run_workflow_template (by name)
```

Two independently runnable roles, each fronted by a playbook and an AAP job template. A separate
config-as-code layer stands up the AAP objects.

### Deployment order

1. Run **EDA config** first → produces the event stream endpoint URL.
2. Paste that URL into the **ServiceNow config** inputs and run it.
3. Build the Flow Designer flow manually and link it on the catalog item's Process Engine tab.

## Components

### Collections — `collections/requirements.yml`
- `servicenow.itsm` — ServiceNow config via the `api` module.
- `ansible.eda` — EDA credential, credential_type, project, rulebook_activation, event_stream.
- `ansible.platform` — config-as-code for AAP gateway/controller objects.

### Role 1 — `snow_catalog_config` (ServiceNow side)
Idempotent (query-then-create/update) tasks using `servicenow.itsm.api`:

- **`outbound_rest.yml`** — `sys_rest_message` (endpoint = EDA event stream URL) + a POST
  `sys_rest_message_fn` function, with HTTP request headers `Authorization` (shared token) and
  `Content-Type: application/json`.
- **`business_rule.yml`** — `sys_script` record on `sc_req_item`, runs after update, with filter
  conditions to prevent loops and scope triggers:
  - Requested for = configured test user
  - Stage = `Request Approved`
  - State = `Work in Progress`
  - Updated by **is not** the service account
  The advanced script comes from a Jinja2 template `business_rule_script.js.j2`; the REST message
  name on line 2 (`new sn_ws.RESTMessageV2('<rest_message_name>', 'post')`) is parameterized to
  the message created above — no hardcoded names.
- **`catalog_item.yml`** — `sc_cat_item` with category, `active=true`, published state.

Flow Designer is **not** automated — see `docs/flow_designer_manual_steps.md`.

### Role 2 — `eda_snow_config` (EDA side)
Uses `ansible.eda`:
- `credential_type` — custom ServiceNow event-stream token type.
- `credential` — the shared token value (supplied at runtime, never in git).
- `event_stream` — created against the organization; its generated URL is pasted into the SNOW
  Outbound REST message.
- `project` + `rulebook_activation` — point at this repo's `rulebooks/servicenow_catalog.yml`.

**Risk / fallback:** if the installed `ansible.eda` version lacks the `event_stream` module, fall
back to a documented UI step plus `ansible.builtin.uri` calls against the EDA Controller API. To be
confirmed during implementation.

### Rulebook — `rulebooks/servicenow_catalog.yml`
Webhook source fed by the event stream. Rules match on payload fields (e.g. `short_description`)
and fire `run_workflow_template` / `run_job_template` **by name** (config-as-code friendly).

### Config-as-code — `configure_aap/` (via `ansible.platform`)
Creates:
- Two **custom credential types**: ServiceNow connection (host/username/password) and shared
  event-stream token.
- **Project** pointing at this repo.
- **Job templates** `Configure ServiceNow Catalog` and `Configure EDA`, each with a **survey** for
  the non-secret variables.
- EDA **project** and **rulebook activation**.

### Secrets / auth flow
The same token string lives in (a) the SNOW Outbound REST `Authorization` header and (b) the EDA
credential. Both injected via AAP custom credentials at launch. ServiceNow instance URL / user /
password injected via the ServiceNow custom credential type.

### `README.md`
Beginner-facing, end to end: ordering sequence diagram, prerequisites, deployment order (EDA first,
then ServiceNow), the manual Flow Designer steps, and how to bootstrap AAP from `configure_aap/`.

## Proposed Repository Layout

```
snow_sc/
├── README.md
├── CLAUDE.md
├── ansible.cfg
├── collections/
│   └── requirements.yml
├── roles/
│   ├── snow_catalog_config/
│   │   ├── defaults/main.yml
│   │   ├── tasks/{main,outbound_rest,business_rule,catalog_item}.yml
│   │   ├── templates/business_rule_script.js.j2
│   │   └── meta/main.yml
│   └── eda_snow_config/
│       ├── defaults/main.yml
│       ├── tasks/main.yml
│       └── meta/main.yml
├── playbooks/
│   ├── configure_servicenow.yml
│   └── configure_eda.yml
├── rulebooks/
│   └── servicenow_catalog.yml
├── configure_aap/
│   ├── README.md
│   ├── credential_types.yml
│   ├── projects.yml
│   ├── job_templates.yml
│   └── eda.yml
└── docs/
    └── flow_designer_manual_steps.md
```

## Data Flow

1. User orders the catalog item → Flow Designer flow runs the approval and sets the Requested Item
   to `Work in Progress` once approved.
2. The Business Rule on `sc_req_item` fires on that update (filters prevent loops and restrict to
   the right user/stage/state), executing the Outbound REST Message.
3. The REST message POSTs a JSON payload (built from the `sc_req_item` record) with the shared token
   to the EDA event stream endpoint.
4. EDA receives the event, the rulebook matches on payload fields, and runs the named job/workflow
   template in the Controller.

## Idempotency & Error Handling

- All ServiceNow `api` tasks query for an existing record (by unique name/sys fields) before
  create/update so reruns converge rather than duplicate.
- EDA `ansible.eda` modules are declarative/idempotent by `state` + `name`.
- Roles fail fast with clear messages when required inputs (instance URL, token, endpoint) are
  missing, via `ansible.builtin.assert`.

## Testing

- `ansible-lint` and `yamllint` clean.
- Idempotency check: second run reports zero changes for both roles.
- Functional smoke test (manual / documented): order the catalog item, approve, confirm the EDA
  activation receives the event and launches the named template.

## Alternative Considered

**Ship a ServiceNow Update Set XML** (including the flow) and import it via Ansible. Rejected: the
flow would be opaque/pre-baked and less reviewable. Keeping the REST message, business rule, and
catalog item as declarative Ansible tasks — with the flow as guided manual steps — is more
transparent and easier for a new user to adapt.
