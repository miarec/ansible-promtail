# ansible-promtail
ansible role to install promtail as a service for log aggregation locally detailed [here](https://grafana.com/docs/loki/latest/installation/local/)

## Overview
 - Promtail Release will be:
    - Downloaded from github repository
    - Inflated
    - Renamed and moved to `/usr/local/bin`
    - Made executable
 - Config File will be created
 - Version File will be creaed
 - SystemD service will be
    - created
    - started
    - enabled

## Role Variables
The following variables define how promtail reports to loki

Example using IP address, loki server at `1.2.3.4:3100`
```
loki_url: "1.2.3.4"             # IP address of the Loki Server
loki_http_listen_port: 3100     # TCP port Loki server is listening
loki_grpc_port: 0               # GRPC port Loki server listens, 0 means a random port
```
Example using FQDN, loki server at `loki.example.com:3105`
```
loki_dns_host: "loki"             # hostname portion of Loki FQDN
loki_dns_domain: "example.com"    # hostname portion of Loki FQDN
loki_http_listen_port: 3105       # TCP port Loki server is listening
```

The following variable defines what promtail will report to loki
```
scrape_jobs: [
  {'job_name':'apache', 'path':'/var/log/apache2/*', }]
```

- `job_name` = unique name for the scrape job
- `path`     = location of log files or directory to scrape

## Example Playbook

Install Promtail with custom labels
```yaml
- hosts: server

  vars:
    loki_url: "10.0.0.10"
    loki_http_listen_port: 3100
    promtail_http_listen_port: 9110
    promtail_custom_environment_variables:
      - "CUSTOM_EV=VALUE1"
    promtail_external_labels:
      static_label: "Static label"

  roles:
    - role: miarec.promtail
```

Install Promtail installed in AWS
```yaml
- hosts: server

  vars:
    loki_url: "10.0.0.10"
    loki_http_listen_port: 3100
    promtail_http_listen_port: 9110
    promtail_aws_imdsv1_data:
      EC2_INSTANCE_ID: 'instance-id'
      EC2_TAG_NAME: 'tags/instance/Name'
      EC2_TAG_ENVIRONMENT: 'tags/instance/Environment'
      EC2_TAG_ROLE: 'tags/instance/Role'
    promtail_external_labels:
      host: "${EC2_TAG_NAME}.${EC2_TAG_ENVIRONMENT}"
      role: "${EC2_TAG_ROLE}"

  roles:
    - role: miarec.promtail
```

## CICD
A Github Action is configured to test this module everytime there is a push to the master branch or on pull requests

The following items are tested
- `yamllint` to verify proper yaml syntax
- `ansible-lint` to verify proper ansible syntax
- `molecule` testing
- Deploy playbook against docker containers with following distros
```
matrix:
  distro:
   - centos7
   - centos8
   - ubuntu1804
   - ubuntu2004
```
- Run `testinfra` testing against each container for the following
  - all directories are present
  - all files are present
  - promtail service is running and enabled
  - server is listening on the configured port
