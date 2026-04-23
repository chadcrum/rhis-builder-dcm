# rhis-builder-dcm

Deploy [DCM (Data Collection Manager)](https://github.com/dcm-project) as a systemd-managed podman quadlet service stack on RHEL 9 or Fedora.

DCM is a microservice platform for managing service providers, catalogs, policies, and placement across infrastructure targets. This repo provides an Ansible role that deploys all DCM services as individual quadlet containers with systemd integration — automatic restarts, dependency ordering, journal logging, and lifecycle management.

## Stack Components

| Service | Image | Description |
|---------|-------|-------------|
| postgres | `docker.io/library/postgres:16-alpine` | Shared PostgreSQL database (4 databases) |
| nats | `docker.io/library/nats:2-alpine` | NATS message broker with JetStream |
| service-provider-manager | `quay.io/dcm-project/service-provider-manager` | Manages service provider registrations |
| catalog-manager | `quay.io/dcm-project/catalog-manager` | Service type catalog and instances |
| policy-manager | `quay.io/dcm-project/policy-manager` | Policy evaluation engine |
| placement-manager | `quay.io/dcm-project/placement-manager` | Resource placement decisions |
| gateway | `docker.io/traefik:v3.4` | Traefik reverse proxy (API gateway) |

### Optional Service Providers

| Service | Image | Description |
|---------|-------|-------------|
| kubevirt-service-provider | `quay.io/dcm-project/kubevirt-service-provider` | KubeVirt VM management |
| k8s-container-service-provider | `quay.io/dcm-project/k8s-container-service-provider` | Kubernetes container workloads |
| acm-cluster-service-provider | `quay.io/dcm-project/acm-cluster-service-provider` | ACM cluster lifecycle |

## Architecture

All services run as standalone containers on a shared bridge network (`dcm-network`). Each container is an independent systemd unit with explicit dependency ordering:

```
dcm-network-network.service
  ├── dcm-postgres.service
  │     ├── dcm-service-provider-manager.service  (+ dcm-nats.service)
  │     ├── dcm-catalog-manager.service
  │     ├── dcm-policy-manager.service
  │     └── dcm-placement-manager.service
  ├── dcm-nats.service
  └── dcm-gateway.service  (after all 4 managers)
        ├── dcm-kubevirt-service-provider.service  (optional)
        ├── dcm-k8s-container-service-provider.service  (optional)
        └── dcm-acm-cluster-service-provider.service  (optional)
```

Container names match the upstream `compose.yaml` exactly (no prefix). Systemd unit names use a `dcm-` prefix. Services resolve each other by container name via Podman DNS.

Configuration files (Traefik routes, PostgreSQL init SQL) are sourced from the [api-gateway](https://github.com/dcm-project/api-gateway) repository, cloned at deploy time. This keeps api-gateway as the single source of truth.

## Prerequisites

- RHEL 9 or Fedora target host with Podman 4.4+ (quadlet support)
- Ansible 2.15+ on the control node
- Network access to pull container images from `docker.io` and `quay.io`
- Dedicated host (container names like `postgres` and `nats` assume no collisions)

## Deployment Phases

The `dcm_deploy` role executes in six phases:

1. **Prerequisites** — installs container tools, firewalld, and git; opens the gateway port; creates config directories; clones the api-gateway repo
2. **Generate configs** — templates the shared environment file; copies Traefik config and PostgreSQL init SQL from the cloned api-gateway repo
3. **Deploy quadlet files** — places `.container`, `.network`, and `.volume` unit files into `/etc/containers/systemd/` and reloads systemd
4. **Initialize database** — starts PostgreSQL, waits for readiness, creates the four manager databases if they don't exist
5. **Start services** — phased startup: NATS, then all four managers (with health checks), then the gateway, then optional providers
6. **Validate** — checks the Traefik `/ping` endpoint, verifies all manager health endpoints through the gateway, and asserts all expected containers are running

## Usage

### Basic deployment

```bash
ansible-playbook -i inventory/hosts \
  --extra-vars "vars_path=vars/dcm.yml" \
  main.yml
```

### With vault secrets

```bash
ansible-playbook -i inventory/hosts \
  --ask-vault-password \
  --extra-vars "vault_path=vault/dcm-vault.yml" \
  --extra-vars "vars_path=vars/dcm.yml" \
  main.yml
```

### Run a specific phase

```bash
ansible-playbook -i inventory/hosts \
  --extra-vars "role_name=dcm_deploy" \
  --extra-vars "task_name=validate_deployment" \
  run_role_task.yml
```

### Enable optional providers

```bash
ansible-playbook -i inventory/hosts \
  --extra-vars "vars_path=vars/dcm.yml" \
  --extra-vars '{"dcm_provider_kubevirt": true, "dcm_kubevirt_kubeconfig": "/path/to/kubeconfig"}' \
  main.yml
```

## Variable Reference

### Container Images

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_postgres_image` | `docker.io/library/postgres` | PostgreSQL image |
| `dcm_postgres_image_tag` | `16-alpine` | PostgreSQL tag |
| `dcm_nats_image` | `docker.io/library/nats` | NATS image |
| `dcm_nats_image_tag` | `2-alpine` | NATS tag |
| `dcm_traefik_image` | `docker.io/traefik` | Traefik image |
| `dcm_traefik_image_tag` | `v3.4` | Traefik tag |
| `dcm_image_registry` | `quay.io/dcm-project` | Registry for DCM manager images |

### Manager Versions

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_service_provider_manager_version` | `main` | Service provider manager image tag |
| `dcm_catalog_manager_version` | `main` | Catalog manager image tag |
| `dcm_policy_manager_version` | `main` | Policy manager image tag |
| `dcm_placement_manager_version` | `main` | Placement manager image tag |

### Database

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_db_user` | `admin` | PostgreSQL username |
| `dcm_db_password` | `adminpass` | PostgreSQL password (override for production) |

### Networking

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_gateway_port` | `9080` | Traefik gateway port published to host |
| `dcm_postgres_port` | `5432` | PostgreSQL port published to host |
| `dcm_nats_port` | `4222` | NATS client port published to host |
| `dcm_nats_monitor_port` | `8222` | NATS monitoring port published to host |

### Paths

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_config_dir` | `/srv/containers/dcm/config` | Configuration files on target host |
| `dcm_quadlet_dir` | `/etc/containers/systemd` | Quadlet unit file directory |

### API Gateway Source

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_api_gateway_repo` | `https://github.com/dcm-project/api-gateway.git` | Repo cloned for config files |
| `dcm_api_gateway_version` | `main` | Branch/tag to clone |

### Feature Toggles

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_provider_kubevirt` | `false` | Enable KubeVirt service provider |
| `dcm_provider_k8s_container` | `false` | Enable K8s container service provider |
| `dcm_provider_acm_cluster` | `false` | Enable ACM cluster service provider |

### KubeVirt Provider

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_kubevirt_provider_name` | `kubevirt-service-provider` | Provider registration name |
| `dcm_kubevirt_namespace` | `default` | Kubernetes namespace |
| `dcm_kubevirt_kubeconfig` | `""` | Path to kubeconfig on target host |

### K8s Container Provider

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_k8s_container_sp_name` | `k8s-container-provider` | Provider registration name |
| `dcm_k8s_container_sp_namespace` | `default` | Kubernetes namespace |
| `dcm_k8s_container_sp_external_svc_type` | `NodePort` | Kubernetes service type for external access |
| `dcm_k8s_container_sp_kubeconfig` | `""` | Path to kubeconfig on target host |

### ACM Cluster Provider

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_acm_cluster_sp_name` | `acm-cluster-sp` | Provider registration name |
| `dcm_acm_cluster_sp_namespace` | `default` | ACM namespace |
| `dcm_acm_cluster_sp_base_domain` | `""` | Base domain for cluster provisioning |
| `dcm_acm_cluster_sp_pull_secret` | `""` | Pull secret for cluster provisioning |
| `dcm_acm_cluster_sp_default_infra_env` | `""` | Default InfraEnv (BareMetal platform) |
| `dcm_acm_cluster_sp_agent_namespace` | `""` | Agent namespace |
| `dcm_acm_cluster_sp_kubeconfig` | `""` | Path to kubeconfig on target host |

### Firewall

| Variable | Default | Description |
|----------|---------|-------------|
| `dcm_firewall_zone` | `public` | Firewalld zone for published ports |

## Verification

After deployment, verify the stack is healthy:

```bash
# Traefik gateway responds
curl http://<host>:9080/ping

# Manager health endpoints (through gateway)
curl http://<host>:9080/api/v1alpha1/health/providers
curl http://<host>:9080/api/v1alpha1/health/catalog
curl http://<host>:9080/api/v1alpha1/health/policies
curl http://<host>:9080/api/v1alpha1/health/placement

# All systemd units active
systemctl status dcm-*.service

# All containers running
podman ps

# Databases created
podman exec postgres psql -U admin -l
```

## Compose Alignment

A standalone playbook `verify_compose_alignment.yml` checks that every core service in the upstream `compose.yaml` has a corresponding quadlet template. Run it in CI to detect drift:

```bash
ansible-playbook verify_compose_alignment.yml
```

## Related Repositories

- [dcm-project/api-gateway](https://github.com/dcm-project/api-gateway) — upstream compose file, Traefik config, and init SQL
- [brianredbeard/rhis-builder-quay](https://github.com/brianredbeard/rhis-builder-quay) — upstream fork origin (Quay quadlet deployment)
