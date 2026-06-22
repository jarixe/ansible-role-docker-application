# Ansible Role: Docker Application

Deploys a Docker Compose application with minimal per-app Ansible code.

## What This Role Does

- Creates the application directories needed on the host.
- Renders `docker-compose.yml` from a template.
- Optionally renders `.env` from `templates/dotenv.j2` when that file exists.
- Optionally renders extra templates for app config files.
- Pulls newer images and reconciles the Compose project on each run.
- Prunes dangling Docker images after deployment.
- Optionally includes playbook-local pre and post task files.

## Requirements

- Docker Engine and the Docker Compose v2 plugin must already be installed on the target host.
- The `community.docker` Ansible collection must be available.
- A Compose template must exist at `templates/docker-compose.yml.j2` beside the playbook, unless you override `docker_application_compose_template`.
- If you want a dotenv file deployed automatically, create `templates/dotenv.j2` beside the playbook or override `docker_application_dotenv_template`.

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `docker_application_name` | `""` | Name of the application. Required. |
| `docker_application_user` | `docker_application_name` | Owner of writable application directories on the host. |
| `docker_application_group` | `docker_application_user` | Group of writable application directories on the host. |
| `docker_application_compose_dir` | `/opt/{{ docker_application_name }}` | Docker Compose project directory. |
| `docker_application_compose_file` | `<compose_dir>/docker-compose.yml` | Destination path for the rendered Compose file. |
| `docker_application_dotenv_file` | `<compose_dir>/.env` | Destination path for the rendered dotenv file. |
| `docker_application_config_dir` | `<compose_dir>/config` | Default writable config directory created by the role. |
| `docker_application_storage_dirs` | `[]` | Additional writable directories to create (storage mounts, subdirectories, etc.). |
| `docker_application_compose_template` | `<playbook_dir>/templates/docker-compose.yml.j2` | Path to the Compose template on the controller. |
| `docker_application_dotenv_template` | `<playbook_dir>/templates/dotenv.j2` | Path to the dotenv template on the controller. |
| `docker_application_deploy_dotenv` | `true` | Deploys `.env` when the dotenv template exists. |
| `docker_application_template_files` | `[]` | Extra templates to render. Each item should define `src` and `dest`, with optional `owner`, `group`, and `mode`. |
| `docker_application_pre_tasks_file` | `<playbook_dir>/tasks/pre.yml` | Optional playbook-local tasks file included before deployment if it exists. |
| `docker_application_post_tasks_file` | `<playbook_dir>/tasks/post.yml` | Optional playbook-local tasks file included after deployment if it exists. |
| `docker_application_pull_policy` | `always` | Compose pull policy used during reconciliation. |
| `docker_application_remove_orphans` | `true` | Removes orphaned services from the Compose project. |
| `docker_application_prune_dangling_images` | `true` | Prunes dangling images after deployment. |

## Example Playbook

```yaml
---
- hosts: uptime-kuma
  become: true
  gather_facts: true
  vars_files:
    - ../../secrets.yml

  vars:
    docker_user: "uptime-kuma"
    docker_uid: "1500"
    docker_gid: "1500"
    docker_storage_dir: "/media/{{ docker_user }}"

  roles:
    - role: ansible-role-base
      vars:
        base_users_extra:
          - name: "{{ docker_user }}"
            uid: "{{ docker_uid }}"
            gid: "{{ docker_gid }}"
        base_packages_extra:
          - python3-docker
      tags: ["base"]

    - role: ansible-role-docker
      vars:
        docker_users:
          - "{{ docker_user }}"
      tags: ["docker"]

    - role: ansible-role-docker-application
      vars:
        docker_application_name: "{{ docker_user }}"
        docker_application_storage_dirs:
          - "{{ docker_storage_dir }}"
      tags: ["uptime-kuma"]
```

## Notes

- The role deliberately supports playbook-local `templates/` and optional `tasks/pre.yml` and `tasks/post.yml`, so simple apps do not need a dedicated role.
- `templates/dotenv.j2` is optional; if it exists and `docker_application_deploy_dotenv` is `true`, the role renders it to `.env` in the Compose directory.
- Custom handlers are not loaded from the playbook directory. If you need app-specific handler behavior, either notify the generic handlers from pre/post tasks or create a dedicated application role.
