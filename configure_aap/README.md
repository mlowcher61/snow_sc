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
