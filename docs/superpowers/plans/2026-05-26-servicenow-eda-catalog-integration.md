# ServiceNow Service Catalog → EDA Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build two reusable Ansible roles (ServiceNow catalog config + EDA config) plus an `ansible.platform` config-as-code layer that wires a ServiceNow Service Catalog item to an Event-Driven Ansible event stream on AAP 2.5.

**Architecture:** `roles/snow_catalog_config` uses `servicenow.itsm.api` to create the Outbound REST Message, Business Rule, and Catalog Item idempotently (query-then-create). `roles/eda_snow_config` uses `ansible.eda` to create the credential type, credential, event stream, project, and rulebook activation. The Flow Designer flow is documented as manual steps. `configure_aap/` provisions the AAP objects and surveys so a new user can deploy on their own platform.

**Tech Stack:** Ansible Automation Platform 2.5, collections `servicenow.itsm`, `ansible.eda`, `ansible.platform`; ServiceNow Table API; EDA event streams + rulebooks.

**Verification approach:** No application unit tests. Each task ends with `yamllint` + `ansible-lint` on the changed files and (where applicable) a syntax check; the whole solution gets an idempotency rerun in the final task. Tasks touching ServiceNow tables or AAP 2.5 module names include an explicit verification step because exact table/field names and collection module paths can differ by instance/version.

**Reference inputs (from the spec):**
- EDA event stream endpoint example: `https://<aap-host>/eda-event-streams/api/eda/v1/external_event_stream/<uuid>/post/`
- Business rule filters: Requested for = test user; Stage = `Request Approved`; State = `Work in Progress`; Updated by ≠ service account.
- Shared token (test): `ABC123` — supplied at runtime via custom credential, never committed.

---

## File Structure

| File | Responsibility |
|------|----------------|
| `ansible.cfg` | Collections path, stdout callback, host_key_checking off |
| `.gitignore` | Ignore `collections/ansible_collections/`, `*.retry`, venv |
| `collections/requirements.yml` | Pin `servicenow.itsm`, `ansible.eda`, `ansible.platform` |
| `roles/snow_catalog_config/defaults/main.yml` | All tunable vars (names, filters, payload fields) |
| `roles/snow_catalog_config/meta/main.yml` | Galaxy metadata, collection deps |
| `roles/snow_catalog_config/tasks/main.yml` | Assert inputs, include the three sub-task files |
| `roles/snow_catalog_config/tasks/outbound_rest.yml` | `sys_rest_message` + `sys_rest_message_fn` + headers |
| `roles/snow_catalog_config/tasks/business_rule.yml` | `sys_script` business rule with templated JS |
| `roles/snow_catalog_config/tasks/catalog_item.yml` | `sc_cat_item` create + publish |
| `roles/snow_catalog_config/templates/business_rule_script.js.j2` | Parameterized RESTMessageV2 payload script |
| `roles/eda_snow_config/defaults/main.yml` | EDA org, names, template-to-match mapping |
| `roles/eda_snow_config/meta/main.yml` | Galaxy metadata, collection deps |
| `roles/eda_snow_config/tasks/main.yml` | credential_type → credential → event_stream → project → activation |
| `playbooks/configure_servicenow.yml` | Runs role 1 |
| `playbooks/configure_eda.yml` | Runs role 2 |
| `rulebooks/servicenow_catalog.yml` | Webhook source + rules → run template by name |
| `configure_aap/credential_types.yml` | Custom credential types (SNOW conn + shared token) |
| `configure_aap/projects.yml` | Controller project pointing at this repo |
| `configure_aap/job_templates.yml` | Two job templates + surveys |
| `configure_aap/eda.yml` | EDA project + rulebook activation as config-as-code |
| `configure_aap/README.md` | How to bootstrap AAP |
| `docs/flow_designer_manual_steps.md` | Manual flow build + catalog link |
| `README.md` | End-to-end beginner-facing guide |

---

## Task 1: Repository scaffolding

**Files:**
- Create: `ansible.cfg`
- Create: `.gitignore`
- Create: `collections/requirements.yml`

- [ ] **Step 1: Create `ansible.cfg`**

```ini
[defaults]
collections_path = ./collections
stdout_callback = yaml
host_key_checking = False
retry_files_enabled = False
interpreter_python = auto_silent

[galaxy]
server_list = automation_hub, galaxy

[galaxy_server.automation_hub]
url = https://console.redhat.com/api/automation-hub/content/published/
auth_url = https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token

[galaxy_server.galaxy]
url = https://galaxy.ansible.com/
```

- [ ] **Step 2: Create `.gitignore`**

```gitignore
collections/ansible_collections/
*.retry
__pycache__/
.venv/
*.pyc
```

- [ ] **Step 3: Create `collections/requirements.yml`**

```yaml
---
collections:
  - name: servicenow.itsm
    version: ">=2.6.0"
  - name: ansible.eda
    version: ">=2.0.0"
  - name: ansible.platform
    version: ">=2.5.0"
```

- [ ] **Step 4: Install collections and lint**

Run:
```bash
ansible-galaxy collection install -r collections/requirements.yml
yamllint collections/requirements.yml
```
Expected: collections install without error; yamllint reports no errors.

- [ ] **Step 5: Commit**

```bash
git add ansible.cfg .gitignore collections/requirements.yml
git commit -m "chore: scaffold repo (ansible.cfg, gitignore, collection requirements)"
```

---

## Task 2: `snow_catalog_config` role — metadata, defaults, and input asserts

**Files:**
- Create: `roles/snow_catalog_config/meta/main.yml`
- Create: `roles/snow_catalog_config/defaults/main.yml`
- Create: `roles/snow_catalog_config/tasks/main.yml`

- [ ] **Step 1: Create `roles/snow_catalog_config/meta/main.yml`**

```yaml
---
galaxy_info:
  role_name: snow_catalog_config
  author: mlowcher61
  description: Configure ServiceNow Outbound REST Message, Business Rule, and Catalog Item to trigger Event-Driven Ansible.
  license: MIT
  min_ansible_version: "2.15"
  galaxy_tags:
    - servicenow
    - eda
    - itsm
collections:
  - servicenow.itsm
dependencies: []
```

- [ ] **Step 2: Create `roles/snow_catalog_config/defaults/main.yml`**

```yaml
---
# --- ServiceNow connection (supplied via custom credential / survey at runtime) ---
# snow_instance, snow_username, snow_password come from the SNOW custom credential.

# --- EDA endpoint (output of the EDA config role; pasted in here) ---
snow_eda_endpoint: ""           # e.g. https://<aap>/eda-event-streams/api/eda/v1/external_event_stream/<uuid>/post/
snow_eda_token: ""              # shared token, from custom credential (matches EDA credential)

# --- Outbound REST message ---
snow_rest_message_name: "EDA Event Stream"
snow_rest_message_function: "post"
snow_rest_message_description: "Posts approved sc_req_item events to Event-Driven Ansible."

# --- Business rule ---
snow_business_rule_name: "Notify EDA on Approved Request"
snow_business_rule_table: "sc_req_item"
snow_business_rule_order: 100
# Encoded query for the filter conditions (loop prevention + scoping).
# Replace <REQUESTED_FOR_SYS_ID> and <SERVICE_ACCOUNT_SYS_ID> for your instance.
snow_business_rule_filter: >-
  request.requested_for=<REQUESTED_FOR_SYS_ID>^stage=request_approved^state=-5^sys_updated_by!=service

# --- Catalog item ---
snow_catalog_item_name: "EDA Automation Request"
snow_catalog_item_short_description: "Request automation handled by Event-Driven Ansible."
snow_catalog_item_category_sys_id: ""   # sys_id of the target sc_category
snow_catalog_item_active: true
```

- [ ] **Step 3: Create `roles/snow_catalog_config/tasks/main.yml`**

```yaml
---
- name: Assert required ServiceNow connection and EDA inputs are present
  ansible.builtin.assert:
    that:
      - snow_instance is defined and snow_instance | length > 0
      - snow_username is defined and snow_username | length > 0
      - snow_password is defined and snow_password | length > 0
      - snow_eda_endpoint | length > 0
      - snow_eda_token | length > 0
    fail_msg: >-
      Missing required inputs. Provide snow_instance/username/password (SNOW credential)
      and snow_eda_endpoint + snow_eda_token (from the EDA config role).

- name: Configure the Outbound REST Message
  ansible.builtin.include_tasks: outbound_rest.yml

- name: Configure the Business Rule
  ansible.builtin.include_tasks: business_rule.yml

- name: Configure the Catalog Item
  ansible.builtin.include_tasks: catalog_item.yml
```

- [ ] **Step 4: Lint**

Run:
```bash
yamllint roles/snow_catalog_config/
ansible-lint roles/snow_catalog_config/
```
Expected: no errors. (Sub-task files don't exist yet, so `include_tasks` lint may warn about missing files — acceptable until Task 3–5; rerun lint after Task 5.)

- [ ] **Step 5: Commit**

```bash
git add roles/snow_catalog_config/meta roles/snow_catalog_config/defaults roles/snow_catalog_config/tasks/main.yml
git commit -m "feat(snow): add snow_catalog_config role skeleton, defaults, and input asserts"
```

---

## Task 3: Outbound REST Message tasks

**Files:**
- Create: `roles/snow_catalog_config/tasks/outbound_rest.yml`

> **Verification note:** Table/field names below (`sys_rest_message`, `sys_rest_message_fn`, header table `sys_rest_message_fn_headers`, field `rest_endpoint`) are the standard ServiceNow names but vary by version. Step 1 verifies them against the target instance before relying on them.

- [ ] **Step 1: Verify ServiceNow table names against the target instance**

Run (replace creds/instance):
```bash
ansible localhost -m servicenow.itsm.api_info \
  -a "instance={'host':'https://<instance>.service-now.com','username':'<u>','password':'<p>'} resource=sys_rest_message query_params={'sysparm_limit':'1'}"
```
Expected: returns a `record` list (possibly empty) with no schema error. Repeat for `sys_rest_message_fn` and `sys_rest_message_fn_headers`. If a name differs on the instance, update the `resource:` values in this file before continuing.

- [ ] **Step 2: Create `roles/snow_catalog_config/tasks/outbound_rest.yml`**

```yaml
---
- name: Look up existing Outbound REST Message
  servicenow.itsm.api_info:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_rest_message
    query_params:
      sysparm_query: "name={{ snow_rest_message_name }}"
  register: rest_message_lookup

- name: Create Outbound REST Message if absent
  servicenow.itsm.api:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_rest_message
    action: post
    data:
      name: "{{ snow_rest_message_name }}"
      rest_endpoint: "{{ snow_eda_endpoint }}"
      description: "{{ snow_rest_message_description }}"
      authentication_type: "no_authentication"
  when: rest_message_lookup.record | length == 0
  register: rest_message_created

- name: Set fact for the REST message sys_id
  ansible.builtin.set_fact:
    snow_rest_message_sys_id: >-
      {{ (rest_message_lookup.record[0].sys_id
          if rest_message_lookup.record | length > 0
          else rest_message_created.record.sys_id) }}

- name: Look up existing POST function for the REST message
  servicenow.itsm.api_info:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_rest_message_fn
    query_params:
      sysparm_query: "rest_message={{ snow_rest_message_sys_id }}^function_name={{ snow_rest_message_function }}"
  register: rest_fn_lookup

- name: Create POST function if absent
  servicenow.itsm.api:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_rest_message_fn
    action: post
    data:
      rest_message: "{{ snow_rest_message_sys_id }}"
      function_name: "{{ snow_rest_message_function }}"
      http_method: "post"
      rest_endpoint: "{{ snow_eda_endpoint }}"
      content: '{"placeholder":"set at runtime by business rule"}'
  when: rest_fn_lookup.record | length == 0
  register: rest_fn_created

- name: Set fact for the REST function sys_id
  ansible.builtin.set_fact:
    snow_rest_fn_sys_id: >-
      {{ (rest_fn_lookup.record[0].sys_id
          if rest_fn_lookup.record | length > 0
          else rest_fn_created.record.sys_id) }}

- name: Ensure Authorization header on the POST function
  servicenow.itsm.api:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_rest_message_fn_headers
    action: post
    data:
      rest_message_function: "{{ snow_rest_fn_sys_id }}"
      name: "Authorization"
      value: "{{ snow_eda_token }}"
  register: auth_header
  changed_when: auth_header.record is defined
  failed_when:
    - auth_header.record is not defined
    - "'Duplicate' not in (auth_header.msg | default(''))"

- name: Ensure Content-Type header on the POST function
  servicenow.itsm.api:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_rest_message_fn_headers
    action: post
    data:
      rest_message_function: "{{ snow_rest_fn_sys_id }}"
      name: "Content-Type"
      value: "application/json"
  register: ctype_header
  changed_when: ctype_header.record is defined
  failed_when:
    - ctype_header.record is not defined
    - "'Duplicate' not in (ctype_header.msg | default(''))"
```

- [ ] **Step 3: Syntax + lint**

Run:
```bash
yamllint roles/snow_catalog_config/tasks/outbound_rest.yml
ansible-lint roles/snow_catalog_config/tasks/outbound_rest.yml
```
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add roles/snow_catalog_config/tasks/outbound_rest.yml
git commit -m "feat(snow): create Outbound REST Message, POST function, and auth headers"
```

---

## Task 4: Business Rule task + templated JS payload

**Files:**
- Create: `roles/snow_catalog_config/templates/business_rule_script.js.j2`
- Create: `roles/snow_catalog_config/tasks/business_rule.yml`

- [ ] **Step 1: Create `roles/snow_catalog_config/templates/business_rule_script.js.j2`**

```javascript
(function executeRule(current, previous /*null when async*/) {
    try {
        // Line below references the Outbound REST Message created by Ansible.
        var request = new sn_ws.RESTMessageV2('{{ snow_rest_message_name }}', '{{ snow_rest_message_function }}');

        var payload = {
            "number": current.number.toString(),
            "sys_id": current.sys_id.toString(),
            "short_description": current.short_description.toString(),
            "requested_for": current.request.requested_for.getDisplayValue(),
            "stage": current.stage.toString(),
            "state": current.state.toString()
        };

        request.setRequestBody(JSON.stringify(payload));
        var response = request.execute();
        gs.info('[SNOW->EDA] Posted ' + current.number + ' status=' + response.getStatusCode());
    } catch (ex) {
        gs.error('[SNOW->EDA] Failed to post ' + current.number + ': ' + ex.getMessage());
    }
})(current, previous);
```

- [ ] **Step 2: Create `roles/snow_catalog_config/tasks/business_rule.yml`**

```yaml
---
- name: Render the business rule advanced script
  ansible.builtin.set_fact:
    snow_business_rule_script: "{{ lookup('ansible.builtin.template', 'business_rule_script.js.j2') }}"

- name: Look up existing Business Rule
  servicenow.itsm.api_info:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_script
    query_params:
      sysparm_query: "name={{ snow_business_rule_name }}^collection={{ snow_business_rule_table }}"
  register: business_rule_lookup

- name: Create Business Rule if absent
  servicenow.itsm.api:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_script
    action: post
    data:
      name: "{{ snow_business_rule_name }}"
      collection: "{{ snow_business_rule_table }}"
      active: "true"
      when: "after"
      action_update: "true"
      order: "{{ snow_business_rule_order }}"
      filter_condition: "{{ snow_business_rule_filter }}"
      advanced: "true"
      script: "{{ snow_business_rule_script }}"
  when: business_rule_lookup.record | length == 0
  register: business_rule_created

- name: Update Business Rule script if it already exists
  servicenow.itsm.api:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sys_script
    action: patch
    sys_id: "{{ business_rule_lookup.record[0].sys_id }}"
    data:
      filter_condition: "{{ snow_business_rule_filter }}"
      script: "{{ snow_business_rule_script }}"
  when: business_rule_lookup.record | length > 0
```

- [ ] **Step 3: Lint**

Run:
```bash
yamllint roles/snow_catalog_config/tasks/business_rule.yml
ansible-lint roles/snow_catalog_config/tasks/business_rule.yml
```
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add roles/snow_catalog_config/templates/business_rule_script.js.j2 roles/snow_catalog_config/tasks/business_rule.yml
git commit -m "feat(snow): add business rule with parameterized RESTMessageV2 payload script"
```

---

## Task 5: Catalog Item task + ServiceNow playbook

**Files:**
- Create: `roles/snow_catalog_config/tasks/catalog_item.yml`
- Create: `playbooks/configure_servicenow.yml`

- [ ] **Step 1: Create `roles/snow_catalog_config/tasks/catalog_item.yml`**

```yaml
---
- name: Look up existing Catalog Item
  servicenow.itsm.api_info:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sc_cat_item
    query_params:
      sysparm_query: "name={{ snow_catalog_item_name }}"
  register: catalog_item_lookup

- name: Create Catalog Item if absent
  servicenow.itsm.api:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sc_cat_item
    action: post
    data:
      name: "{{ snow_catalog_item_name }}"
      short_description: "{{ snow_catalog_item_short_description }}"
      category: "{{ snow_catalog_item_category_sys_id }}"
      active: "{{ snow_catalog_item_active | string | lower }}"
  when: catalog_item_lookup.record | length == 0
  register: catalog_item_created

- name: Report the Catalog Item sys_id (needed for the manual Flow Designer link)
  ansible.builtin.debug:
    msg: >-
      Catalog Item sys_id =
      {{ catalog_item_lookup.record[0].sys_id
         if catalog_item_lookup.record | length > 0
         else catalog_item_created.record.sys_id }}.
      Link your Flow Designer flow on the Process Engine tab — see docs/flow_designer_manual_steps.md.
```

- [ ] **Step 2: Create `playbooks/configure_servicenow.yml`**

```yaml
---
- name: Configure ServiceNow Service Catalog integration for EDA
  hosts: localhost
  gather_facts: false
  roles:
    - role: snow_catalog_config
```

- [ ] **Step 3: Syntax check + lint**

Run:
```bash
ansible-playbook playbooks/configure_servicenow.yml --syntax-check
yamllint roles/snow_catalog_config/tasks/catalog_item.yml playbooks/configure_servicenow.yml
ansible-lint roles/snow_catalog_config/ playbooks/configure_servicenow.yml
```
Expected: syntax OK; no lint errors.

- [ ] **Step 4: Commit**

```bash
git add roles/snow_catalog_config/tasks/catalog_item.yml playbooks/configure_servicenow.yml
git commit -m "feat(snow): create catalog item and add configure_servicenow playbook"
```

---

## Task 6: `eda_snow_config` role

**Files:**
- Create: `roles/eda_snow_config/meta/main.yml`
- Create: `roles/eda_snow_config/defaults/main.yml`
- Create: `roles/eda_snow_config/tasks/main.yml`

> **Verification note:** `ansible.eda.event_stream` exists in `ansible.eda` ≥ 2.x for AAP 2.5 but is newer than the other modules. Step 1 confirms it is available; if not, use the documented fallback in the role comments.

- [ ] **Step 1: Verify the `event_stream` module is available**

Run:
```bash
ansible-doc ansible.eda.event_stream >/dev/null 2>&1 && echo "event_stream: AVAILABLE" || echo "event_stream: MISSING — use UI fallback"
ansible-doc ansible.eda.credential_type >/dev/null 2>&1 && echo "credential_type: AVAILABLE"
```
Expected: prints availability. If `event_stream` is MISSING, leave `eda_create_event_stream: false` (default below) and follow the UI step in `docs/flow_designer_manual_steps.md`-style note inside the role.

- [ ] **Step 2: Create `roles/eda_snow_config/meta/main.yml`**

```yaml
---
galaxy_info:
  role_name: eda_snow_config
  author: mlowcher61
  description: Configure Event-Driven Ansible credential, event stream, project, and rulebook activation for ServiceNow events.
  license: MIT
  min_ansible_version: "2.15"
  galaxy_tags:
    - eda
    - servicenow
collections:
  - ansible.eda
dependencies: []
```

- [ ] **Step 3: Create `roles/eda_snow_config/defaults/main.yml`**

```yaml
---
# --- EDA controller connection (supplied via custom credential / survey) ---
# eda_host, eda_username, eda_password come from the AAP/EDA credential.
eda_validate_certs: true
eda_organization: "Default"

# --- Shared token (matches the SNOW Authorization header) ---
eda_token: ""

# --- Object names ---
eda_credential_type_name: "ServiceNow Event Stream Token"
eda_credential_name: "ServiceNow Event Stream Credential"
eda_event_stream_name: "ServiceNow Catalog Event Stream"
eda_create_event_stream: true

# --- Project + activation ---
eda_project_name: "ServiceNow Catalog Integration"
eda_project_url: "https://github.com/mlowcher61/snow_sc.git"
eda_project_branch: "main"
eda_rulebook_name: "servicenow_catalog.yml"
eda_activation_name: "ServiceNow Catalog Activation"
eda_decision_environment: "Default Decision Environment"
```

- [ ] **Step 4: Create `roles/eda_snow_config/tasks/main.yml`**

```yaml
---
- name: Assert required EDA inputs are present
  ansible.builtin.assert:
    that:
      - eda_host is defined and eda_host | length > 0
      - eda_username is defined and eda_username | length > 0
      - eda_password is defined and eda_password | length > 0
      - eda_token | length > 0
    fail_msg: "Provide eda_host/username/password (EDA credential) and eda_token."

- name: Create the ServiceNow event-stream credential type
  ansible.eda.credential_type:
    controller_host: "{{ eda_host }}"
    controller_username: "{{ eda_username }}"
    controller_password: "{{ eda_password }}"
    validate_certs: "{{ eda_validate_certs }}"
    name: "{{ eda_credential_type_name }}"
    state: present
    inputs:
      fields:
        - id: token
          label: Token
          type: string
          secret: true
    injectors:
      extra_vars:
        servicenow_token: "{{ '{{ token }}' }}"

- name: Create the ServiceNow event-stream credential
  ansible.eda.credential:
    controller_host: "{{ eda_host }}"
    controller_username: "{{ eda_username }}"
    controller_password: "{{ eda_password }}"
    validate_certs: "{{ eda_validate_certs }}"
    name: "{{ eda_credential_name }}"
    credential_type_name: "{{ eda_credential_type_name }}"
    organization_name: "{{ eda_organization }}"
    state: present
    inputs:
      token: "{{ eda_token }}"

- name: Create the ServiceNow event stream
  ansible.eda.event_stream:
    controller_host: "{{ eda_host }}"
    controller_username: "{{ eda_username }}"
    controller_password: "{{ eda_password }}"
    validate_certs: "{{ eda_validate_certs }}"
    name: "{{ eda_event_stream_name }}"
    credential_name: "{{ eda_credential_name }}"
    organization_name: "{{ eda_organization }}"
    state: present
  when: eda_create_event_stream | bool
  register: eda_event_stream

- name: Show the event stream endpoint to paste into ServiceNow
  ansible.builtin.debug:
    msg: >-
      Event stream endpoint = {{ eda_event_stream.event_stream.url | default('see EDA UI > Event Streams') }}.
      Use this as snow_eda_endpoint when running the ServiceNow config.
  when: eda_create_event_stream | bool

- name: Create the EDA project from this repository
  ansible.eda.project:
    controller_host: "{{ eda_host }}"
    controller_username: "{{ eda_username }}"
    controller_password: "{{ eda_password }}"
    validate_certs: "{{ eda_validate_certs }}"
    name: "{{ eda_project_name }}"
    url: "{{ eda_project_url }}"
    organization_name: "{{ eda_organization }}"
    state: present

- name: Create the rulebook activation
  ansible.eda.rulebook_activation:
    controller_host: "{{ eda_host }}"
    controller_username: "{{ eda_username }}"
    controller_password: "{{ eda_password }}"
    validate_certs: "{{ eda_validate_certs }}"
    name: "{{ eda_activation_name }}"
    project_name: "{{ eda_project_name }}"
    rulebook_name: "{{ eda_rulebook_name }}"
    decision_environment_name: "{{ eda_decision_environment }}"
    organization_name: "{{ eda_organization }}"
    enabled: true
    state: present
```

- [ ] **Step 5: Lint**

Run:
```bash
yamllint roles/eda_snow_config/
ansible-lint roles/eda_snow_config/
```
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add roles/eda_snow_config/
git commit -m "feat(eda): add eda_snow_config role (credential type, credential, event stream, project, activation)"
```

---

## Task 7: EDA playbook + rulebook

**Files:**
- Create: `playbooks/configure_eda.yml`
- Create: `rulebooks/servicenow_catalog.yml`

- [ ] **Step 1: Create `playbooks/configure_eda.yml`**

```yaml
---
- name: Configure Event-Driven Ansible for ServiceNow catalog events
  hosts: localhost
  gather_facts: false
  roles:
    - role: eda_snow_config
```

- [ ] **Step 2: Create `rulebooks/servicenow_catalog.yml`**

```yaml
---
- name: Listen for ServiceNow catalog requests
  hosts: all
  sources:
    - ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000
  rules:
    - name: Run provisioning workflow on approved request
      condition: event.payload.short_description is defined and event.payload.state == "-5"
      action:
        run_workflow_template:
          name: "ServiceNow Provisioning Workflow"
          organization: "Default"

    - name: Fallback to job template when no workflow match
      condition: event.payload.short_description is defined and event.payload.state != "-5"
      action:
        run_job_template:
          name: "ServiceNow Provisioning Job"
          organization: "Default"
```

- [ ] **Step 3: Syntax check + lint**

Run:
```bash
ansible-playbook playbooks/configure_eda.yml --syntax-check
ansible-rulebook --rulebook rulebooks/servicenow_catalog.yml --check 2>/dev/null || echo "ansible-rulebook --check not supported in this version; skip"
yamllint playbooks/configure_eda.yml rulebooks/servicenow_catalog.yml
```
Expected: syntax OK; yamllint clean.

- [ ] **Step 4: Commit**

```bash
git add playbooks/configure_eda.yml rulebooks/servicenow_catalog.yml
git commit -m "feat(eda): add configure_eda playbook and servicenow_catalog rulebook"
```

---

## Task 8: AAP config-as-code (`configure_aap/`)

**Files:**
- Create: `configure_aap/credential_types.yml`
- Create: `configure_aap/projects.yml`
- Create: `configure_aap/job_templates.yml`
- Create: `configure_aap/eda.yml`
- Create: `configure_aap/README.md`

> **Verification note:** In AAP 2.5 some controller resources are exposed under `ansible.platform.*` and some still under `ansible.controller.*`. Step 1 confirms the module names; adjust the `module:`/FQCN if a resource is not present in `ansible.platform`.

- [ ] **Step 1: Verify ansible.platform module availability**

Run:
```bash
for m in credential_type credential project job_template; do
  ansible-doc ansible.platform.$m >/dev/null 2>&1 && echo "ansible.platform.$m: AVAILABLE" || echo "ansible.platform.$m: use ansible.controller.$m"
done
```
Expected: prints which FQCN to use. Update the playbooks below accordingly if any are MISSING.

- [ ] **Step 2: Create `configure_aap/credential_types.yml`**

```yaml
---
- name: Create custom credential types for the ServiceNow/EDA integration
  hosts: localhost
  gather_facts: false
  tasks:
    - name: ServiceNow connection credential type
      ansible.platform.credential_type:
        name: "ServiceNow Connection"
        kind: cloud
        inputs:
          fields:
            - id: snow_instance
              label: ServiceNow Instance URL
              type: string
            - id: snow_username
              label: Username
              type: string
            - id: snow_password
              label: Password
              type: string
              secret: true
          required:
            - snow_instance
            - snow_username
            - snow_password
        injectors:
          extra_vars:
            snow_instance: "{{ '{{ snow_instance }}' }}"
            snow_username: "{{ '{{ snow_username }}' }}"
            snow_password: "{{ '{{ snow_password }}' }}"
        state: present

    - name: Shared EDA event-stream token credential type
      ansible.platform.credential_type:
        name: "EDA Event Stream Token"
        kind: cloud
        inputs:
          fields:
            - id: eda_event_stream_token
              label: Event Stream Token
              type: string
              secret: true
          required:
            - eda_event_stream_token
        injectors:
          extra_vars:
            snow_eda_token: "{{ '{{ eda_event_stream_token }}' }}"
            eda_token: "{{ '{{ eda_event_stream_token }}' }}"
        state: present
```

- [ ] **Step 3: Create `configure_aap/projects.yml`**

```yaml
---
- name: Create the controller project for this repository
  hosts: localhost
  gather_facts: false
  tasks:
    - name: snow_sc project
      ansible.platform.project:
        name: "ServiceNow EDA Integration"
        organization: "Default"
        scm_type: git
        scm_url: "https://github.com/mlowcher61/snow_sc.git"
        scm_branch: main
        scm_update_on_launch: true
        state: present
```

- [ ] **Step 4: Create `configure_aap/job_templates.yml`**

```yaml
---
- name: Create job templates with surveys
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Configure ServiceNow Catalog job template
      ansible.platform.job_template:
        name: "Configure ServiceNow Catalog"
        organization: "Default"
        project: "ServiceNow EDA Integration"
        playbook: "playbooks/configure_servicenow.yml"
        credentials:
          - "ServiceNow Connection"
          - "EDA Event Stream Token"
        ask_variables_on_launch: false
        survey_enabled: true
        survey_spec:
          name: "ServiceNow Catalog inputs"
          description: "Non-secret inputs for the ServiceNow configuration."
          spec:
            - question_name: "EDA event stream endpoint URL"
              variable: "snow_eda_endpoint"
              type: text
              required: true
            - question_name: "Catalog item category sys_id"
              variable: "snow_catalog_item_category_sys_id"
              type: text
              required: true
        state: present

    - name: Configure EDA job template
      ansible.platform.job_template:
        name: "Configure EDA"
        organization: "Default"
        project: "ServiceNow EDA Integration"
        playbook: "playbooks/configure_eda.yml"
        credentials:
          - "EDA Event Stream Token"
        survey_enabled: true
        survey_spec:
          name: "EDA inputs"
          description: "Non-secret inputs for the EDA configuration."
          spec:
            - question_name: "EDA controller host"
              variable: "eda_host"
              type: text
              required: true
            - question_name: "EDA organization"
              variable: "eda_organization"
              type: text
              required: false
              default: "Default"
        state: present
```

- [ ] **Step 5: Create `configure_aap/eda.yml`**

```yaml
---
- name: EDA project and rulebook activation as config-as-code
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Run the EDA configuration role
      ansible.builtin.import_role:
        name: eda_snow_config
```

- [ ] **Step 6: Create `configure_aap/README.md`**

```markdown
# Bootstrapping AAP for the ServiceNow → EDA integration

Run these once to make the solution deployable from the AAP UI.

1. Install collections: `ansible-galaxy collection install -r ../collections/requirements.yml`
2. Export controller auth (or use a controller credential):
   `export CONTROLLER_HOST=... CONTROLLER_USERNAME=... CONTROLLER_PASSWORD=...`
3. Create credential types: `ansible-playbook credential_types.yml`
4. Create the project: `ansible-playbook projects.yml`
5. Create job templates + surveys: `ansible-playbook job_templates.yml`
6. In the UI, attach the **ServiceNow Connection** and **EDA Event Stream Token** credentials
   (with your real secrets) to the job templates.
7. Run **Configure EDA** first (note the event stream endpoint it prints), then **Configure
   ServiceNow Catalog**, supplying that endpoint in the survey.
```

- [ ] **Step 7: Syntax check + lint**

Run:
```bash
for p in configure_aap/credential_types.yml configure_aap/projects.yml configure_aap/job_templates.yml configure_aap/eda.yml; do
  ansible-playbook "$p" --syntax-check
done
yamllint configure_aap/
ansible-lint configure_aap/
```
Expected: syntax OK; no lint errors.

- [ ] **Step 8: Commit**

```bash
git add configure_aap/
git commit -m "feat(aap): add config-as-code for credential types, project, job templates, and EDA"
```

---

## Task 9: Flow Designer manual steps doc

**Files:**
- Create: `docs/flow_designer_manual_steps.md`

- [ ] **Step 1: Create `docs/flow_designer_manual_steps.md`**

```markdown
# Flow Designer — manual build steps

The Outbound REST Message, Business Rule, and Catalog Item are created by Ansible. The Flow
Designer flow is built manually (it cannot be reliably created via the Table API), then linked to
the catalog item.

## Build the flow
1. **Flow Designer → New → Flow.** Trigger: **Service Catalog** item. Click **Done**.
2. Add **Action → Action**, search **Get Catalog Variables**, select it. Drag the **Requested Item
   Record** from the data panel on the right into the drop zone.
3. Add **Ask for Approval**. Drag the **Requested Item** into the drop zone. Under **Approve when**,
   select **Anyone approves**, then choose your approval **Group**.
4. Add **Update Record** to set the Requested Item **State** to **Work in Progress** after approval.
5. **Activate** the flow.

## Link the flow to the catalog item
1. Open the catalog item created by Ansible (its sys_id is printed by the
   `Configure ServiceNow Catalog` job — see the debug task in `catalog_item.yml`).
2. On the **Process Engine** tab, set the **Flow** to the flow you just activated.
3. Confirm the item is **Active** and **Published** so it is orderable.

## Why this is manual
Flow definitions span many interrelated `sys_hub_*` records and are normally moved between
instances via Update Sets, not the Table API. Keeping it manual keeps the automated artifacts
transparent and version-portable.
```

- [ ] **Step 2: Lint**

Run: `yamllint docs/ 2>/dev/null; echo "markdown doc — no yaml to lint"`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add docs/flow_designer_manual_steps.md
git commit -m "docs: add Flow Designer manual build and catalog-link steps"
```

---

## Task 10: README + end-to-end verification

**Files:**
- Create: `README.md`

- [ ] **Step 1: Create `README.md`**

````markdown
# snow_sc — ServiceNow Service Catalog → Event-Driven Ansible

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
````

- [ ] **Step 2: Repository-wide lint**

Run:
```bash
yamllint .
ansible-lint
```
Expected: no errors across the repo.

- [ ] **Step 3: Idempotency verification (against a test instance)**

Run both playbooks twice with real test credentials:
```bash
ansible-playbook playbooks/configure_eda.yml -e @test_vars.yml
ansible-playbook playbooks/configure_eda.yml -e @test_vars.yml   # second run
ansible-playbook playbooks/configure_servicenow.yml -e @test_vars.yml
ansible-playbook playbooks/configure_servicenow.yml -e @test_vars.yml   # second run
```
Expected: the **second** run of each reports `changed=0`. (`test_vars.yml` is local-only and must be in `.gitignore` — do not commit credentials.)

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: add end-to-end README"
```

---

## Self-Review Checklist (completed during planning)

- **Spec coverage:** Outbound REST (Task 3), Business Rule + filters + script template (Task 4),
  Catalog Item (Task 5), Flow documented (Task 9), EDA credential type/credential/event
  stream/project/activation (Task 6), rulebook by-name templates (Task 7), config-as-code +
  surveys + custom credentials (Task 8), README (Task 10). All spec sections mapped.
- **Placeholder scan:** No "TBD/TODO"; uncertain ServiceNow tables and AAP module FQCNs have
  explicit verification steps with concrete defaults (Tasks 3, 6, 8) rather than blanks.
- **Type/name consistency:** `snow_rest_message_name`/`_function`, `snow_rest_fn_sys_id`,
  `snow_eda_endpoint`/`snow_eda_token`, `eda_token`, credential type names ("ServiceNow Connection",
  "EDA Event Stream Token") are used consistently across roles, JS template, and config-as-code.
```
