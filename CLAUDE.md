# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the OpenStack K8S Operators Architectures repository containing Kustomize templates for validated architectures (VA) and deployment topologies (DT) for OpenStack on Kubernetes/OpenShift.

## Key Commands

### Testing and Validation

```bash
# Validate all kustomization files can be built
./test-kustomizations.sh

# Validate specific directories
./test-kustomizations.sh dt/bgp va/hci

# Validate YAML schema for automation files
yamale -s .ci/automation-schema.yaml automation/vars
```

### CI Job Management

```bash
# Regenerate Zuul CI jobs after adding new scenarios
./create-zuul-jobs.py
```

### Documentation

```bash
# Set up and serve documentation locally
python3 -m venv local/docs-venv
source local/docs-venv/bin/activate
pip install -r docs/doc_requirements.txt
mkdocs serve
```

### Testing with CI Framework

```bash
# Run architecture tests locally
cd ci-framework
make run_ctx_architecture_test \
    SCENARIO_NAME=your_scenario \
    ARCH_REPO=../architecture \
    NET_ENV_FILE=./ci/playbooks/files/networking-env-definition.yml
```

## Architecture and Structure

### Directory Layout

- **`/lib/`** - Base templates shared across all architectures (control planes, data planes, networking, storage)
- **`/va/`** - Validated Architecture templates for production deployments
- **`/dt/`** - Deployment Topology templates for testing environments
- **`/examples/`** - User-customizable environment templates
- **`/automation/`** - CI/CD variables and mock data defining deployment stages
- **`/zuul.d/`** - Zuul CI configuration (auto-generated, do not edit directly)

### Key Patterns

1. **Kustomize Components**: All configurations use Kustomize v5.0.1+ for templating
2. **Layered Architecture**: Base → VA/DT specific → User environment
3. **Stage-based Deployments**: Multiple stages defined in automation/vars/
4. **DRY Principle**: Common configurations in lib/, specialized in va/dt/

### Important Development Notes

1. **Multi-stage Deployments**: Always reference base kustomizations in later stages to avoid accidental OpenStackControlPlane overwrites
2. **Kustomization Organization**: Keep all kustomizations for a resource in one file
3. **CI Job Creation**: Add mock data to `automation/mocks/` for new scenarios, then run `create-zuul-jobs.py`
4. **PR Testing**: Use CI-Framework reproducer with custom parameters for validation
5. **Cherry-picking**: Use `/cherrypick branch-name` comment or manual git cherry-pick for stable branches

### Testing Requirements

- **Kustomize** 5.0.1+ or **OpenShift CLI** 4.14+
- **Python** with pyyaml for CI job generation
- **Podman** for CI Framework container testing

## NodeNetworkConfigurationPolicy (NNCP) Structure

### Current Repository Structure

The repository uses NNCP with the following pattern:
- Base templates in `/lib/nncp/` with minimal metadata
- Template file (`ocp_node_template.yaml`) with placeholders
- Kustomize replacements from `network-values` ConfigMap
- Supports IPv4, IPv6, and L3-only configurations

### RHOSO 18.0 Requirements

Based on RHOSO 18.0 documentation, NNCPs should follow these conventions:

1. **Interface Naming**: 
   - VLAN interfaces: Can use descriptive names (e.g., `internalapi`) or traditional format (e.g., `enp1s0.20`)
   - Control plane interface names must be consistent across all nodes

2. **Required Fields**:
   - `reorder-headers: true` for VLAN interfaces (missing in current templates)
   - Explicit interface descriptions
   - Each worker node requires exactly 1 IP address per interface

3. **Critical Architecture Difference - Direct Interface vs Bridge**:
   - **Current Repository**: Uses linux-bridge (`ospbr`) for control plane with IP assigned to bridge
   - **RHOSO 18.0**: Uses direct ethernet interface with IP assigned to physical interface
   - **Impact**: 94 references to `ospbr` across 38 files need migration to direct interface approach

4. **Structure Differences**:
   - RHOSO 18.0 examples show more direct configuration without heavy templating
   - Interface names in RHOSO docs use physical interface names (e.g., `enp1s0.20`) vs logical names (`internalapi`)

### Network Configuration Structure

The `network-values` ConfigMap contains:
- **Node-specific data**: IP addresses for each network per node
- **Network definitions**: CIDR, VLAN IDs, MTU, interface names
- **DNS and routing**: DNS resolver config and route definitions
- **Bridge configuration**: Bridge names and settings