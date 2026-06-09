# aap_config — AAP Configuration-as-Code

Stands up every AAP object the ServiceNow → Event-Driven Ansible solution needs, on **your**
platform, in one idempotent run. Built on the certified **`infra.aap_configuration`** collection:
you declare *what* should exist in `files/*.yml` data structures, and the `dispatch` role
reconciles AAP to match. Re-running changes nothing already correct.

## Layout

```
aap_config/
├── load.yml          # entry point — applies files/ via dispatch, then runs validate.yml
├── validate.yml      # post-load check — fails if any expected object is missing
├── requirements.yml  # collections (infra.aap_configuration, ansible.platform, ansible.eda)
├── inventory/
│   └── aap.yml       # localhost — the CaC run talks to AAP over the API
├── group_vars/
│   └── all.yml       # connection, object names, secret inputs (all via env vars)
└── files/
    ├── gateway_organizations.yml      # organization + Galaxy credentials
    ├── controller_credential_types.yml # ServiceNow Connection + EDA Controller Connection + Event Stream Token
    ├── controller_credentials.yml      # the three credentials, filled from env vars
    ├── controller_projects.yml         # this repo as a controller project
    ├── controller_inventories.yml      # localhost inventory used by the job templates
    ├── controller_hosts.yml            # explicit localhost host with local connection
    ├── controller_job_templates.yml    # Smoke tests + Configure EDA + Configure ServiceNow Catalog (+ surveys)
    └── eda_projects.yml                # this repo as an EDA project (rulebook source)
```

## No secrets in git

Every sensitive value is resolved at run time from an environment variable (see
`group_vars/all.yml`) and written into AAP **custom credentials** attached to the job templates.
Nothing secret is committed, surveyed, or vaulted in this repo.

## Run it

1. Install the collections:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```
   This repo pins `ansible.controller` to `4.7.10` because newer controller
   collection releases may reject the legacy `controller_oauthtoken` alias
   still used internally by `infra.aap_configuration`.
2. Create a personal API token in the AAP UI, then export the connection + secrets:
   ```bash
   export AAP_HOSTNAME=https://<aap-host>
   export AAP_TOKEN=<personal token from the AAP UI>
   export SNOW_INSTANCE=https://<inst>.service-now.com
   export SNOW_USERNAME=admin SNOW_PASSWORD='***'
   export EDA_HOSTNAME=https://<aap-host>
   export EDA_USERNAME=admin EDA_PASSWORD='***'
   export EDA_EVENT_STREAM_TOKEN='***'        # same token both sides share
   # optional: AAP_ORGANIZATION (default "Default"), SNOW_SC_SCM_URL, SNOW_SC_SCM_BRANCH
   ```
3. Apply, then auto-validate:
   ```bash
   ansible-playbook -i inventory/ load.yml
   ```

## After it runs

1. Launch **AAP Localhost Smoke Test** first. If this fails with no output, the issue is in AAP execution, not the solution playbooks.
2. Launch **AAP EDA Credential Smoke Test** next. If this fails with no output, the issue is in EDA credential injection or launch setup.
3. Launch **Configure EDA** next — note the event stream endpoint it prints.
4. Launch **Configure ServiceNow Catalog**, supplying that endpoint in the survey.
5. Build and link the Flow Designer flow: see `../docs/flow_designer_manual_steps.md`.
6. Order the catalog item, approve it, and confirm the rulebook activation runs your template.

## Customizing

Object **names** and the **repo URL/branch** live in `group_vars/all.yml`. The **shape** of each
object (fields, survey questions, credentials attached) lives in the matching `files/*.yml`. Edit
the data file, re-run `load.yml`, and `dispatch` reconciles the change.
