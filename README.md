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

 - `promtail_loki_url` define where promtail should send log, this can be a FQDN or IP address

    Examples
    ```
    promtail_loki_url = "https://loki.example.com"
    ```

    ```
    promtail_loki_url = "http://1.1.1.1:3100"
    ```

### Promtail configuration variables
The following variable defines what promtail will report to loki

 - `promtail_scrape_jobs` defines what log files promatil will track
    - `job_name` = name for the scrape job
    - `path` = location of log files or directory to scrape, wildcards are allowed

    Example
    ```
    promtail_scrape_jobs: [
      {'job_name':'apache', 'path':'/var/log/apache2/*', }]
    ```
 - `promtail_custom_environment_variables` custom envrionment variables that can be added to promtail environment file

 - `promtail_external_labels` this will be attached to all metrics on all jobs delivered to loki
    - `key` will be the label name supplied to loki, it would be a good idea to use standard lables for all instance monitored
    - `value` data that will be supplied, this can be a test string or a enviromental variable that can be read when loki is reloaded

    Example
    ```yaml
      promtail_external_labels:
        host: "${EC2_TAG_NAME}.${EC2_TAG_ENVIRONMENT}"
        role: "${EC2_TAG_ROLE}"
        custom: "custom ssting that will be applied to all logs sent to loki"
    ```

### AWS Labels

 - `promtail_aws_install` when `true`, instructs the role to query instance metadata from AWS IMDS, default = `false`
 - `promtail_aws_imdsv1_data` This dictonary defines what metadata should be queried from AWS IMDS version1 and how it is written to the promtail environment file,
    - `key` is the name of the metric, this will be the environment variable used in `promtail_external_labels`
    - `value` is the path that should be queried, more information in the [AWS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-categories.html)

    Example
    ```yaml
        promtail_aws_imdsv1_data:
          EC2_INSTANCE_ID: 'instance-id'
    ```
    The result of `curl -sL http://169.254.169.254/latest/meta-data/instance-id` will be saved with key `EC2_INSTANCE_ID` and will be wriiten to `/etc/promtail/promtail-env`
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
