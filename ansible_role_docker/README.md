# ansible-role-docker

Installs Docker CE on Hosts, configures the deamon and initates a database backup job 

## Requirements

Ansible Version 2.9

## Role Variables

```yaml
##################
# OS preparation #
##################

docker_obsolete_packages:
  - docker
  - docker-engine
  - docker.io
  - docker-doc
  - docker-compose
  - docker-compose-v2
  - podman-docker
  - containerd

docker_necessary_packages:
  - apt-transport-https
  - ca-certificates
  - gnupg2
  - grep
  - zstd
  - mawk
  - curl
  - software-properties-common
  - python3-docker

#########################
# Install configuration #
#########################

docker_install_buildx_plugin: true
docker_install_compose_plugin: true
docker_daemon_options: {}
docker_logins: []

############################
# Optional database backup #
############################

docker_database_backup_enable: false
docker_database_backup_path: "/opt/docker-database-backup"
docker_database_backup_cron_name: "run docker database backup"
docker_database_backup_cron_minute: "{{ 59 | random(seed=inventory_hostname) }}"
docker_database_backup_cron_hour: "12,21"
```

## Dependencies

[Community General Collection](https://galaxy.ansible.com/community/general?sc_cid=701f2000001OH7YAAW) (comes with `ansible`, but not with `ansible-core`)

## Example Playbook

```yaml
---
- hosts: docker_host_group
  vars:
    docker_logins:
      - docker_registry_url: "registry.example.com"
        docker_registry_user: "example.user"
        docker_registry_pass: "password" # ensure to encrypt your credentials using ansible vault
    docker_daemon_options:
      ipv6: true
      fixed-cidr-v6: fd00:111::/64
      experimental: true
      ip6tables: true
      registry-mirrors:
        - "https://mirror1.example.com"
        - "https://mirror2.example.com"
      bip: "10.50.0.1/16"
      default-address-pools:
        - base: "10.51.0.1/16"
          size: 24
      icc: false
      no-new-privileges: true
      # userns-remap: default
      log-level: info
      log-driver: "gelf"
      log-opts:
        gelf-address: "udp://graylog-target.example.com:12201"
        labels: "{{ ansible_hostname }}"
        tag: "{{ inventory_hostname }}/{%raw%}{{.ImageName}}/{{.Name}}/{{.ID}}{%endraw%}"
  roles:
    - role: fw-oss.docker
      become: true
```

## License

MIT

## Author Information

FW-OSS, 2024
