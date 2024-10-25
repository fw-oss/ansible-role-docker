# ansible-role-docker

Installs Docker CE on Hosts, configures the deamon and initates a database backup job.

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
docker_database_backup_mysql_regex: "(mariadb|mysql)"
docker_database_backup_postgres_regex: "(postgres|mattermost-docker-database).*(?<!zammad-backup_1)$"
docker_database_backup_mongo_regex: "(mongo[^-])"
docker_database_backup_keep_regex: "7"
```

## Dependencies

[Community General Collection](https://docs.ansible.com/ansible/latest/collections/community/general/index.html) (comes with `ansible`, but not with `ansible-core`)

## Docker Daemon Configuration
You can configure the docker daemon by setting the `docker_daemon_options` variable. The given dict is translated into a json file and placed at `/etc/docker/daemon.json`. You can find further information about the available options in the [docker documentation](https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file). Additional security best practices can be found in the [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html).

### Good practices
Here are some good practices for configuring the docker daemon. Be aware, that the list is just an indicative, non-exhaustive list and should be adapted to your needs.
- `bip: "10.50.0.1/16"`: Set the bridge IPv4 subnet, choose any unused from `10.0.0.0/8`, `172.16.0.0/12` or `192.168.0.0/16`. See the sample below.
- `default-address-pools`: Set the default address pools for the bridge and the overlay networks to allow more than 31 networks. Choose any unused from `10.0.0.0/8`, `172.16.0.0/12` or `192.168.0.0/16`. See the sample below.
- `icc: false`: Disable inter-container communication through [docker0 bridge network](https://docs.docker.com/network/drivers/bridge/), which means that only [containers in the same network can communicate](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html#rule-5-be-mindful-of-inter-container-connectivity) with each other.
- `no-new-privileges: true`: Prevent [in-container privilege escalation](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html#rule-4-prevent-in-container-privilege-escalation). This prevents container to gain new privileges via `setuid` or `setgid`.
- `userns-remap: default`: Enable [user namespace remapping](https://docs.docker.com/engine/security/userns-remap/#enable-userns-remap-on-the-daemon) to ensure that containers don't run as root on the host. **Be careful since a change in this setting will cause all existing containers to be removed!**
- `log-level: info`: Set the [log level](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html#rule-10-keep-the-docker-daemon-logging-level-at-info) to info.

### IPv6 Configuration
You need to set [some options](https://docs.docker.com/config/daemon/ipv6/) to allow the usage of IPv6:
- `experimental: true`: Enable experimental features - this is required to use IPv6 for the time being.
- `ipv6: true`:  Enable IPv6.
- `fixed-cidr-v6: "fd00:111::/64"`: Set the fixed CIDR to your IPv6 subnet.
- `ip6tables: true`: Enable IPv6 packet filter rules for network isolation and port mapping.

- If you have an additional subnet routed to your host, you can add it to the `default-address-pools` list. See the sample below.

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
        - base: "fd00:112::/64",
          size: 80
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
