# Ansible Best Practices Analysis

Analysis of `rhis-builder-nbde` repository against [Red Hat Automation Good Practices](https://redhat-cop.github.io/automation-good-practices/)

**Analysis Date:** 2026-04-29  
**Total YAML Lines:** 316

---

## ✅ Strengths

### File Naming & Structure
- ✅ All files use `.yml` extension (not `.yaml`)
- ✅ Role name `tang_container` uses underscore (no dashes - good for collection compatibility)
- ✅ Tasks use descriptive imperative names ("Ensure...", "Configure...")
- ✅ Host-specific variables properly organized in `host_vars/<hostname>/` directories

### Code Quality
- ✅ Consistent YAML formatting and indentation
- ✅ Uses FQCN (Fully Qualified Collection Names) for modules: `ansible.builtin.*`, `containers.podman.*`, etc.
- ✅ Good use of `default(omit)` for optional parameters
- ✅ Proper use of `loop_control` with custom `loop_var`
- ✅ Includes LICENSE file (GPL-3.0)
- ✅ Uses ansible-lint directives (`# noqa: risky-shell-pipe`)

### Security
- ✅ Vault-based credential management (registry credentials, encryption passwords)
- ✅ Vault files referenced via external variable (`vault_path`) rather than hardcoded

---

## ⚠️ Issues Found

### CRITICAL

#### 1. Missing Role Documentation (`tang_container`)
**Issue:** No `README.md` in role  
**Best Practice:** "Include meaningful README with examples, inputs, outputs, and rollback capabilities"  
**Impact:** Difficult for users to understand role purpose, inputs, and usage

**Recommendation:**
```
roles/tang_container/README.md should document:
- Role purpose and functionality
- Required variables (containers list structure)
- Optional variables and defaults
- Example playbook usage
- Supported platforms
- Dependencies
- Idempotency guarantees
```

#### 2. Missing Argument Specifications
**Issue:** No `meta/argument_specs.yml`  
**Best Practice:** "Declare argument specifications in meta/argument_specs.yml"  
**Impact:** No parameter validation, poor error messages, no auto-documentation

**Recommendation:** Create `roles/tang_container/meta/argument_specs.yml` to validate the `containers` list structure

#### 3. Missing Role Defaults
**Issue:** No `defaults/main.yml` (empty directory)  
**Best Practice:** "Use defaults/main.yml for all external inputs with documentation"  
**Impact:** No documented default values, role cannot run without explicit variables

**Recommendation:**
```yaml
# roles/tang_container/defaults/main.yml
---
# List of container definitions to deploy
# See README.md for complete structure documentation
tang_container_containers: []
```

### HIGH PRIORITY

#### 4. Variable Naming - Missing Role Prefix
**Issue:** Role uses unprefixed variables (`containers`, `container`)  
**Best Practice:** "Use snake_case with role-name prefixes to avoid collisions. All defaults and arguments require the pattern rolename_argument"  
**Impact:** Risk of variable name collisions with other roles or playbooks

**Current:**
```yaml
containers:
  - name: "tang"
```

**Should be:**
```yaml
tang_container_containers:
  - name: "tang"
```

**Files affected:**
- `roles/tang_container/tasks/main.yml` (uses `containers`)
- `host_vars/tang1.parmstrong.ca/containers.yml` (defines `containers`)

#### 5. Playbooks Mix roles: Section with tasks: Section
**Issue:** Both `clevis_client.yml` and `containerhost_tang.yml` have a `tasks:` section that includes roles  
**Best Practice:** "Use either roles: section OR tasks: section with import_role/include_role, never both"

**Current:**
```yaml
- name: "Install Tang Container on targets"
  hosts: containerhosts
  tasks:
    - name: "Load the vault variables"
      ansible.builtin.include_vars:
        file: "{{ vault_path }}"
    - name: "Apply role tang"
      ansible.builtin.include_role:
        name: "tang_container"
```

**Recommended pattern:**
```yaml
- name: "Install Tang Container on targets"
  hosts: containerhosts
  vars_files:
    - "{{ vault_path }}"
  roles:
    - tang_container
```

#### 6. Vault Loading in Playbooks (Not Roles)
**Issue:** Vault loading repeated in every playbook  
**Best Practice:** "Keep playbooks minimal; move logic into roles"  
**Impact:** Code duplication, inconsistent patterns

**Recommendation:** Either use `vars_files` at play level or move vault loading into role if needed

#### 7. Missing Role Metadata
**Issue:** No `meta/main.yml`  
**Best Practice:** Roles should include metadata for dependencies, supported platforms, galaxy info  
**Impact:** Cannot declare role dependencies, no platform documentation

**Recommendation:** Create `roles/tang_container/meta/main.yml` with:
```yaml
---
galaxy_info:
  role_name: tang_container
  author: parmstro
  description: Deploy Tang server in Podman container with systemd integration
  license: GPL-3.0
  min_ansible_version: "2.9"
  platforms:
    - name: EL
      versions:
        - "8"
        - "9"

dependencies: []
```

### MEDIUM PRIORITY

#### 8. Inventory Structure
**Issue:** Single-file inventory instead of directory structure  
**Best Practice:** "Create structured inventory directories rather than single files"

**Current:**
```
inventory (single file)
```

**Recommended:**
```
inventory/
├── hosts.yml
├── group_vars/
│   ├── containerhosts/
│   │   └── main.yml
│   └── clevishosts/
│       └── main.yml
└── host_vars/
    ├── tang1.parmstrong.ca/
    │   └── containers.yml
    └── clevis1.parmstrong.ca/
        └── bindings.yml
```

#### 9. No Check Mode Support Indicators
**Issue:** No documentation of check mode support  
**Best Practice:** "Roles should pass check mode on first run"  
**Impact:** Unknown if role supports `--check` mode safely

**Recommendation:** Document in README whether role supports check mode and any limitations

#### 10. Task File Naming Could Be Improved
**Issue:** Task files don't follow `sub | Description` pattern  
**Current:** `ensure_container_tools.yml`, `get_container.yml`  
**Recommended:** `sub_ensure_container_tools.yml`, `sub_get_container.yml`  
**Note:** This is minor and current naming is acceptable

### LOW PRIORITY

#### 11. No Collections Structure
**Issue:** Not organized as an Ansible collection  
**Best Practice:** "Package roles as collections to enable namespace isolation"  
**Impact:** Cannot distribute via Ansible Galaxy as collection, no namespace isolation  
**Recommendation:** Consider migrating to collection structure if planning to share:
```
collections/
└── ansible_collections/
    └── parmstro/
        └── rhis_nbde/
            ├── roles/
            │   └── tang_container/
            ├── galaxy.yml
            └── README.md
```

#### 12. Missing ansible-lint Configuration
**Issue:** No `.ansible-lint` configuration file  
**Best Practice:** Use ansible-lint for automated code quality checks  
**Recommendation:** Add `.ansible-lint` to enable automated linting

#### 13. Group Variables Structure
**Issue:** `group_vars/empty.yml` is a placeholder with no real groups  
**Best Practice:** Use group_vars for group-specific variables  
**Recommendation:**
```
group_vars/
├── containerhosts/
│   └── main.yml  # Common container host settings
└── clevishosts/
    └── main.yml  # Common clevis client settings
```

#### 14. No CI/CD Configuration
**Issue:** No GitHub Actions, GitLab CI, or other CI/CD  
**Best Practice:** Automated testing and linting  
**Recommendation:** Add basic ansible-lint and syntax check CI workflow

---

## 📋 Prioritized Action Items

### Phase 1: Critical Documentation & Structure
1. **Create `roles/tang_container/README.md`** - Document role purpose, variables, examples
2. **Create `roles/tang_container/defaults/main.yml`** - Define default values with comments
3. **Create `roles/tang_container/meta/argument_specs.yml`** - Add input validation
4. **Create `roles/tang_container/meta/main.yml`** - Add role metadata

### Phase 2: Variable Naming Refactor
5. **Rename variables to use `tang_container_` prefix**
   - Update role tasks to use `tang_container_containers`
   - Update host_vars files
   - Update documentation

### Phase 3: Playbook Improvements
6. **Refactor playbooks** - Use `roles:` section instead of mixing with `tasks:`
7. **Consolidate vault loading** - Use `vars_files` instead of include_vars in tasks

### Phase 4: Enhanced Structure
8. **Restructure inventory** - Convert to directory-based structure
9. **Add `.ansible-lint` configuration**
10. **Document check mode support**

### Phase 5: Optional Enhancements
11. **Consider collection migration** - If planning to distribute publicly
12. **Add CI/CD** - Automated linting and testing
13. **Restructure group_vars** - Add group-specific defaults

---

## 🎯 Quick Wins (Can do immediately)

1. Add `roles/tang_container/README.md`
2. Add `roles/tang_container/defaults/main.yml` with empty containers list
3. Add `.ansible-lint` configuration
4. Refactor playbooks to use `vars_files` and `roles:` section

---

## 📚 Additional Notes

### What's Working Well
The codebase is clean, follows most YAML style guidelines, uses FQCN, and has good security practices with vault usage. The main issues are around **role documentation** and **variable naming conventions** rather than functional problems.

### Breaking Changes Required
The variable renaming (`containers` → `tang_container_containers`) is a breaking change that requires updating:
- Role task files
- Host variable files  
- Any external references

Consider doing this refactor before wider adoption.

### References
- [Red Hat Automation Good Practices](https://redhat-cop.github.io/automation-good-practices/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Ansible Lint](https://ansible-lint.readthedocs.io/)
