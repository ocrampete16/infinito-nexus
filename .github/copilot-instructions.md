# Infinito.Nexus AI Agent Instructions

## Project Overview
Infinito.Nexus is a modular infrastructure automation framework built on Ansible, Docker, and Linux for deploying self-hosted applications with centralized identity management (LDAP/OIDC). It supports both servers and desktop workstations with a role-based architecture.

## Key Architecture Patterns

### Role-Based Modular System
- Roles follow naming convention: `{category}-{subcategory}-{name}` (e.g., `web-app-nextcloud`, `sys-bkp-provider`)
- Categories defined in `roles/categories.yml` determine role behavior and invokability
- Each role contains standardized structure: `tasks/main.yml`, `defaults/main.yml`, `templates/`, etc.

### Docker Compose Template System
- All web applications use Docker Compose with shared template inheritance
- Base template: `roles/docker-compose/templates/base.yml.j2` provides common services (database, Redis, OAuth2)
- Container templates: `roles/docker-container/templates/` provide reusable components (healthchecks, networks, dependencies)
- Example pattern: `{% include 'roles/docker-container/templates/base.yml.j2' %}`

### CLI Command Structure
- Main entry point: `main.py` with dynamic command discovery from `cli/` directory
- Commands mirror directory structure: `infinito build defaults users` → `cli/build/defaults/users.py`
- All CLI scripts use argparse and can be invoked via `python3 -m cli.{path}.{command}`

## Essential Development Workflows

### Deployment Commands
```bash
# Standard deployment
infinito deploy inventory.yml --limit hostname --host-type server

# Build system before deployment
make messy-build  # Generates applications.yml, users.yml, role includes

# Development deployment
infinito deploy --test --skip-validation --skip-build inventory.yml
```

### Role Development
```bash
# Create new role
infinito create role web-app-myapp --network 172.20.100 --ports http,websocket

# Generate role dependencies graph
infinito build graph -r web-app-nextcloud -D 2

# Validate role structure
infinito validate role web-app-myapp
```

### Build System
- `Makefile` drives the build process with targets like `messy-build`, `clean`, `list`
- Dynamic generation of `group_vars/all/04_applications.yml` from role metadata
- Auto-generated role includes in `tasks/groups/` based on categories

## Configuration Patterns

### Variable Resolution
- Global defaults: `group_vars/all/00_general.yml`
- Application configs use filter plugins: `applications | get_app_conf(application_id, 'images.app', True)`
- Domain mapping: `domains | get_domain(application_id)` and `domains | get_url(application_id, WEB_PROTOCOL)`

### Secrets Management
- Ansible Vault for sensitive data with `infinito vault` commands
- Credential generation: `infinito create credentials --role-path roles/app-name`
- Template pattern: `{{ vault_secrets[application_id].database.password }}`

### Docker Integration
- Service enablement: `applications | is_docker_service_enabled(application_id, 'database')`
- Image resolution: `applications | get_docker_image(application_id, 'web')`
- Port mapping: `ports.localhost.http[application_id]` for consistent port allocation

## Critical Files and Directories

### Core Configuration
- `ansible.cfg`: Custom plugin paths for filters, lookups, modules
- `playbook.yml`: Three-stage execution (constructor → host_type → destructor)
- `group_vars/all/`: Global variable definitions with numbered priority

### Custom Ansible Extensions
- `filter_plugins/`: Data transformation (e.g., `get_app_conf.py`, `domain_utils.py`)
- `lookup_plugins/`: External data sources (e.g., `application_gid.py`)
- `module_utils/`: Shared utilities across custom modules

### Template System
- `roles/docker-compose/templates/base.yml.j2`: Foundation for all Docker Compose files
- `roles/docker-container/templates/`: Reusable container configuration snippets
- Template inheritance pattern enables consistent service definitions

## Common Debugging Patterns

### Inventory Issues
```bash
# Validate inventory structure
infinito validate inventory path/to/inventory/

# Generate full inventory for testing
infinito build inventory full --host localhost --format yaml
```

### Role Testing
```bash
# Test specific application deployment
infinito deploy --limit hostname --id web-app-nextcloud inventory.yml

# Skip validation for faster iteration
infinito deploy --skip-validation --skip-build inventory.yml
```

### Build System Debugging
```bash
# Clean and rebuild
make clean && make messy-build

# Check role categorization
infinito meta categories invokable -s "-"
```

When modifying roles, always consider the template inheritance system and ensure Docker Compose services follow the established patterns. Use the provided filter plugins for consistent variable resolution across the platform.

## Virtual Environment

The virtual environment is managed with `venv`. It has to live outside of this repository. It is located in `~/infinito-venv/`. To activate it, run:

```bash
source ~/infinito-venv/bin/activate
```

## Tests

This repository contains a set of tests. After activating the virtual environment, you can run the tests using:

```bash
# Run all tests
make messy-test

# Run specific test file
python3 -m unittest <path_to_test_file>
```
