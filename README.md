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

### Connection variables
TODO make this a little easier

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

### Promtail configuration variables
The following variable defines what promtail will report to loki

 - `scrape_jobs` defines what log files promatil will track, where `job_name` = defines a name for the scrape job and `path`  defines the location of log files or directory to scrape
example
```
promtail_scrape_jobs: [
  {'job_name':'apache', 'path':'/var/log/apache2/*', }]
```
 - `promtail_custom_environment_variables` custom envrionment variables that can be added to promtail
 - `promtail_external_labels` this will be attached to all metrics on all jobs delivered to loki

### AWS Labels
 - `promtail_aws_imdsv1_data` This dictonary defines what metadata should be queried from AWS IMDS and how it is written to the promtail environment file, the key is the name of the metric and the value is the path that should be queried
 example
```yaml
     promtail_aws_imdsv1_data:
      EC2_INSTANCE_ID: 'instance-id'
```
the result of `curl -sL http://169.254.169.254/latest/meta-data/instance-id` will be save with key `EC2_INSTANCE_ID` and will be wriiten to `/etc/promtail/promtail-env`
```
$ cat /etc/promtail/promtail-env
SYSTEMD_LOG_LEVEL=debug
EC2_INSTANCE_ID=i-02b4a42c6267b8f43
```

Used with `promtail_external_labels`, EC2 metadata can be assigned to all logs forwarded to Loki

## Example Playbook

Install Promtail with custom labels
```yaml
- hosts: server

  vars:
    loki_url: "10.0.0.10"
    loki_http_listen_port: 3100
    promtail_http_listen_port: 9110
    promtail_scrape_jobs: [
        {'job_name':'apache', 'path':'/var/log/apache2/*', },
        {'job_name':'miarec_speech', 'path':'/var/log/miarec_speech/*', }]
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
    promtail_scrape_jobs: [
        {'job_name':'apache', 'path':'/var/log/apache2/*', },
        {'job_name':'miarec_speech', 'path':'/var/log/miarec_speech/*', }]
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
