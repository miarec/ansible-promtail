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
promtail_hostname: "promtail"   # Unique name for Promtail endpoint to be identified by
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
  {'job_name':'{{ promtail_hostname }}-systemd-journal', 'label':'journal', 'path':'/var/log/journal'},
  {'job_name':'{{ promtail_hostname }}-apache', 'label':'apache', 'path':'/var/log/apache2/*', }]
```

- `job_name` = unique name for the scrape job, its recommended to include variable {{ promtail_hostname }} if multiple promtail clients are reporting
- `label`    = label for job, this can be used to organize queries
- `path`     = location of log files or directory to scrape

## Example Playbook

```yaml
- hosts: server

  vars:
    promtail_hostname: "server1"
    loki_url: "10.0.0.1"
    scrape_jobs: [
        {'job_name':'{{ promtail_hostname }}-systemd-journal', 'label':'journal', 'path':'/var/log/journal'},
        {'job_name':'{{ promtail_hostname }}-apache', 'label':'apache', 'path':'/var/log/apache2/*', },
        {'job_name':'{{ promtail_hostname }}-miarec_speech', 'label':'miarec_speech', 'path':'/var/log/miarec_speech/*', }]

  roles:
    - ansible-promtail
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
