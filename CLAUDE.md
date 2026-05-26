# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

This repository is currently empty — no code, git history, or README yet. The directory name `snow_sc` indicates a **ServiceNow Service Catalog ↔ Ansible Automation Platform** integration solution (Service Catalog items triggering AAP job templates / workflows). Update this file with architecture and commands once the solution takes shape.

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
