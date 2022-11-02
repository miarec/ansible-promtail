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

Teh following variables define how promtail sends data to Loki and how it handles errors

Readline variables define how many lines of log file will be read in a cycle
- `promtail_readline_rate_enabled` enables rate limits globally for promtail instance default = `true`
- `promtail_readline_rate` how many lines will be read at a time, default `10000`
- `promtail_readline_rate_drop`When true, exceeding the rate limit causes this instance of Promtail to discard log lines, rather than sending them to Loki, default = `false`

Backoff Configuration vars define error handling when Loki is unavailable
- `promtail_backoff_min_period` defines the ammount of time between retries, when loki is unavailable, default = `"500ms"`
- `promtail_backoff_max_period`defines the total ammount of time promtail will attempt to send data to Loki, default = `"5m"`
- `promtail_backoff_max_retries` defines the total times promtail will attempt to send data to Loki, default = `10`


Batch variables, promtail will wait until either of the following critera are met before packging and sending to Loki, example if 1.04MB of log data is collected in 500ms, then data will be sent every 500ms, if it at 1 second, .5MB of log data is collected, thatn .5MB will be sent every second
- `promtail_batchwait` defines the ammount of time that promatil will wait before sending buffered data to Loki, default = `"1s"`
- `promtail_batchsize` defines the total size of buffered data that promatil will wait to collect before sending to Loki, default = `1048576`

## Example Playbook

```yaml
- hosts: server

  vars:
    promtail_hostname: "server1"
    loki_url: "10.0.0.1"
    scrape_jobs: [
        {'job_name':'apache', 'path':'/var/log/apache2/*', },
        {'job_name':'miarec_speech', 'path':'/var/log/miarec_speech/*', }]

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
