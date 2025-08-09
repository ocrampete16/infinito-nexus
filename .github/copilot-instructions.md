# Infinito.Nexus Copilot Instructions

## Project Overview
Infinito.Nexus is a modular IT infrastructure automation framework built on Ansible, Docker, and Linux. It provides automated deployment, management, and security for self-hosted environments with centralized IAM (LDAP/OIDC), single sign-on, and an extensive application ecosystem.

## Architecture Fundamentals

### Role-Based Structure
- **Roles** in `roles/` follow naming conventions: `{category}-{subcategory}-{name}` (e.g., `web-app-nextcloud`, `sys-bkp-provider`)
- **Categories** defined in `roles/categories.yml` with `invokable` flags determining deployment eligibility
- **Host types**: `server` (default) or `desktop` - affects role selection and variables

### Key Components
- **Entry point**: `main.py` - CLI wrapper with colorized output and command discovery
- **Deployment**: `cli/deploy.py` - orchestrates Ansible playbook execution with modes/validation
- **Build system**: `Makefile` generates dynamic configs (applications.yml, users.yml, role includes)
- **Core playbook**: `playbook.yml` runs 3-stage pipeline: constructor → host_type → destructor

## Critical Workflows

### Deployment Pipeline
```bash
# Standard deployment with validation and build
infinito playbook --host-type server --update inventory.yml

# Development workflow
make messy-build  # Generate dynamic configs
make messy-test   # Run validation suite
cli/deploy.py inventory.yml --skip-validation --verbose
```

### Configuration Generation
- Run `make messy-build` before deployment - generates:
  - `group_vars/all/04_applications.yml` from role discovery
  - `group_vars/all/03_users.yml` from role analysis
  - `tasks/groups/*.yml` role inclusion files

## Custom Ansible Extensions

### Filter Plugins (`filter_plugins/`)
Domain management is central - key filters:
- `get_domain(domains, app_id)` - extracts primary domain from complex domain configs
- `generate_all_domains()` - flattens domain structures, adds www redirects
- `alias_domains_map()` / `canonical_domains_map()` - domain alias handling
- `applications_if_group_and_deps()` - conditional app inclusion based on group membership

### Module Utils (`module_utils/`)
- `domain_utils.py` - shared domain parsing logic used by filters
- `config_utils.py` - application configuration utilities
- `entity_name_utils.py` - entity naming conventions

### Custom Modules (`library/`)
- `cert_check_exists.py` / `cert_folder_find.py` - certificate management
- Used for TLS/Let's Encrypt automation

## Configuration Patterns

### Variable Merging Strategy
Constructor stage (`tasks/stages/01_constructor.yml`) implements complex variable merging:
1. Merge defaults with user configs (recursive combining)
2. Filter applications by group membership and dependencies
3. Generate domain mappings with redirects
4. Create unified `current_play_*` variables for active deployment

### Domain Configuration
Applications support flexible domain configs:
```yaml
domains:
  canonical: "app.example.com"  # Primary domain
  aliases: ["app2.example.com"] # Additional domains
```
Use `{{ domains | get_domain('app-name') }}` in templates.

### Mode System
Deployment supports operational modes via CLI flags:
- `mode_reset`, `mode_test`, `mode_update`, `mode_backup`, `mode_cleanup`
- Set as extra vars, consumed by roles for conditional behavior

## Development Conventions

### Role Development
- Each role should handle all deployment modes (check mode variables)
- Use `when: enable_debug | bool` for debug output
- Service roles prefix: `srv-`, `svc-`, `sys-`, `web-app-`, `desk-`, `dev-`

### Testing
- Unit tests in `tests/unit/` mirror source structure
- Integration test: `cli/integration/deploy_localhost.py` (full Docker deployment)
- Inventory validation: `cli/validate/inventory.py` checks against defaults

### CLI Command Discovery
`main.py` auto-discovers commands in `cli/` subdirectories by scanning for argparse usage. New CLI tools are automatically available via `infinito <command>`.

## Key Files to Reference
- `roles/categories.yml` - role taxonomy and invokability
- `group_vars/all/00_general.yml` - global defaults and deployment settings
- `ansible.cfg` - custom plugin paths
- `requirements.txt` - Python dependencies including `tld` for domain parsing
- `Dockerfile` - containerized development/deployment environment

## Common Patterns
- Import domain utilities: `from module_utils.domain_utils import get_domain`
- Access app config: `{{ applications[app_id] }}` after constructor merge
- Conditional role inclusion: Check `group_names` membership
- Custom filter errors: `raise AnsibleFilterError(f"message: {details}")`
