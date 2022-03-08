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
```
loki_url: "1.1.1.1"             # IP address or FQDN of the Loki Server
loki_http_listen_port: 3100     # TCP port Loki server is listening
loki_grpc_port: 0               # GRPC port Loki server listens, 0 means a random port
promtail_hostname: "promtail"   # Unique name for Promtail endpoint to be identified by
```
The following variable defines what promtail will report to loki
```
scrape_jobs: [
  {'job_name':'{{ promtail_hostname }}-systemd-journal', 'label':'journal', 'path':'/var/log/journal'},
  {'job_name':'{{ promtail_hostname }}-apache', 'label':'apache', 'path':'/var/log/apache2/*', }]
```

- `job_name` = unique name for scrape job, its recommended to include variable {{ promtail_hostname }} for reference
- `label`    = label for job, this can be used to organize queries
- `path`     = location of log files or directory to scrape

## Example Playbook

```yaml
- hosts: server

  vars:
    promtail_version: "2.4.2"
    loki_url: "10.0.0.1"
    scrape_jobs: [
        {'job_name':'{{ promtail_hostname }}-systemd-journal', 'label':'journal', 'path':'/var/log/journal'},
        {'job_name':'{{ promtail_hostname }}-apache', 'label':'apache', 'path':'/var/log/apache2/*', },
        {'job_name':'{{ promtail_hostname }}-miarec_speech', 'label':'miarec_speech', 'path':'/var/log/miarec_speech/*', }]

  roles:
    - ansible-promtail
```

## Whats with the Verison file?
Eagle eyed engineers might have noticed that the version file action is not part of the Grafana Loki documentation

In order to allow for idempotency, not only do we need to check that Loki is install and running, we need to verify the version,  this is a challenge because loki is not installed with a traditional package manager like `apt` or `yum`.

To resolve this, part of the install process is to create a file that contains the installed version,   this file can be read and registered to a variable `__promtail_version` that can be verified against input variable `promtail_version`

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
