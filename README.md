prometheus
==========

Ansible role which installs and configures Prometheus.

The configuration of the role is done in such way that it should not be necessary
to change the role for any kind of configuration. All can be done either by
changing role parameters or by declaring completely new configuration as a
variable. That makes this role absolutely universal. See the examples below for
more details.

Please report any issues or send PR.


Examples
--------

```yaml
---

- name Example of how to use the role with default parameters
  hosts: myhost1
  roles:
    - prometheus

- name Example of how to customize the role
  hosts: myhost2
  vars:
    # Change scraping interval
    prometheus_config_global_evaluation_interval: 15s
    prometheus_config_global_scrape_interval: 30s
    # Add additional scrape_config
    prometheus_config_scrape_configs__custom:
      - job_name: service-z
        tls_config:
          cert_file: valid_cert_file
          key_file: valid_key_file
        bearer_token: mysecret
    # Change hostname and protocol for the Alertmanager
    prometheus_config_alerting_alertmanagers_static_configs_targets__default:
      - myamserver1
    prometheus_config_alerting_alertmanagers_item__custom:
      - scheme: https
    # Define which rule files to load
    prometheus_config_rule_files:
      - /etc/prometheus/rules/*.yml
    # Create rule files
    prometheus_rule_files:
      # This is the name of the file in /etc/prometheus/rules/
      alerting_example1:
        # This is the content of the file
        groups:
          - name: example
            rules:
              - alert: InstanceDown
                expr: up == 0
                for: 5m
                labels:
                  severity: page
                annotations:
                  # We need double 'raw' escaping because the variable is
                  # templated twice, once when read from the playbook, second
                  # time when used in the loop.
                  summary: "{{ '{% raw -%}' }}{% raw -%}Instance {{ $labels.instance }} down{% endraw -%}{{ '{% endraw -%}' }}"
                  # This is another way how to do the escaping - wrap the
                  # curly brackets in the Golang themplating like Jinja2 string.
                  description: "{% raw %}{{ '{{' }} $labels.instance {{ '}}' }} of job {{ '{{' }} $labels.job {{ '}}' }} has been down for more than 5 minutes.{% endraw %}"
  roles:
    - prometheus

- name Example of how to create main config file from scratch
  hosts: myhost3
  vars:
    prometheus_config:
      global:
        evaluation_interval: 20s
        external_labels: {}
        scrape_interval: 20s
        scrape_timeout: 10s
      alerting:
        alertmanagers:
          - static_configs:
              - targets:
                  - alertmanager:9093
      remote_read: []
      remote_write: []
      rule_files: []
      scrape_configs:
        - job_name: prometheus
          static_configs:
            - targets:
                - localhost:9090
  roles:
    - prometheus
```


Role variables
--------------

List of variables used by the role:

```yaml
# YUM repo URL
prometheus_yum_repo_url: "{{
  prometheusio_yum_repo_url |
  default('https://packagecloud.io/prometheus-rpm/release/el/' ~ ansible_facts.distribution_major_version ~ '/$basearch/') }}"

# YUM repo GPG key URL
prometheus_yum_repo_gpgkey: "{{
  prometheusio_yum_repo_gpgkey |
  default([
    'https://packagecloud.io/prometheus-rpm/release/gpgkey',
    'https://raw.githubusercontent.com/lest/prometheus-rpm/master/RPM-GPG-KEY-prometheus-rpm']) }}"

# Additional YUM repo params
prometheus_yum_repo_params: "{{ prometheusio_yum_repo_params | default({})}}"

# APT repo string
prometheus_apt_repo_string: "{{
  prometheusio_apt_repo_string |
  default('deb https://packagecloud.io/prometheus-deb/release/ubuntu xenial main') }}"

# GPG key for the APT repo
prometheus_apt_repo_key: "{{
  prometheusio_apt_repo_key |
  default('https://packagecloud.io/prometheus-deb/release/gpgkey') }}"

# Additional YUM repo params
prometheus_apt_repo_params: "{{ prometheusio_apt_repo_params | default({})}}"

# Package to be installed (explicit version can be specified here)
prometheus_pkg: "{{
  'prometheus2'
    if ansible_facts.os_family == 'RedHat'
    else
  'prometheus' }}"

# Name of the service
prometheus_service: prometheus


# Location of the service default file
prometheus_default_file: /etc/default/prometheus

# Location of the timeseries storage directory
prometheus_storage_dir: /var/lib/prometheus/data

# Default service defaults
prometheus_default__default:
  prometheus_opts: >
    --config.file={{ prometheus_config_file }}
    --storage.tsdb.path={{ prometheus_storage_dir }}

# Custom service defaults
prometheus_default__custom: {}

# Final service defaults
prometheus_default: "{{
  prometheus_default__default | combine(
  prometheus_default__custom) }}"


# Location of the main config file
prometheus_config_file: /etc/prometheus/prometheus.yml


# Values of the default options of the global section
prometheus_config_global_evaluation_interval: 15s
prometheus_config_global_scrape_interval: 15s
prometheus_config_global_scrape_timeout: 10s
prometheus_config_global_external_labels: {}

# Default options of the global section
prometheus_config_global__default:
  evaluation_interval: "{{ prometheus_config_global_evaluation_interval }}"
  scrape_interval: "{{ prometheus_config_global_scrape_interval }}"
  scrape_timeout: "{{ prometheus_config_global_scrape_timeout }}"
  external_labels: "{{ prometheus_config_global_external_labels }}"

# Custom options of the global section
prometheus_config_global__custom: {}

# Final options of the global section
prometheus_config_global: "{{
  prometheus_config_global__default | combine(
  prometheus_config_global__custom) }}"


# Values of the options of the targets subsection of the static_configs
# subsection of the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers_static_configs_targets_host: localhost
prometheus_config_alerting_alertmanagers_static_configs_targets_port: 9093

# Default options of the targets subsection of the static_configs subsection of
# the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers_static_configs_targets__default:
  - "{{
        prometheus_config_alerting_alertmanagers_static_configs_targets_host ~
        ':' ~
        prometheus_config_alerting_alertmanagers_static_configs_targets_port }}"

# Custom options of the targets subsection of the static_configs subsection of
# the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers_static_configs_targets__custom: []

# Final options of the targets subsection of the static_configs subsection of
# the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers_static_configs_targets: "{{
  prometheus_config_alerting_alertmanagers_static_configs_targets__default +
  prometheus_config_alerting_alertmanagers_static_configs_targets__custom }}"


# Default item of the static_configs subsection of the alertmanagers subsection
# of the alerting section
prometheus_config_alerting_alertmanagers_static_configs_item__default:
  targets: "{{ prometheus_config_alerting_alertmanagers_static_configs_targets }}"

# Custom item of the static_configs subsection of the alertmanagers subsection
# of the alerting section
prometheus_config_alerting_alertmanagers_static_configs_item__custom: {}

# Final item of the static_configs subsection of the alertmanagers subsection
# of the alerting section
prometheus_config_alerting_alertmanagers_static_configs_item: "{{
  prometheus_config_alerting_alertmanagers_static_configs_item__default | combine(
  prometheus_config_alerting_alertmanagers_static_configs_item__custom) }}"


# Default options of the static_configs subsection of the alertmanagers
# subsection of the alerting section
prometheus_config_alerting_alertmanagers_static_configs__default:
  - "{{ prometheus_config_alerting_alertmanagers_static_configs_item }}"

# Custom options of the static_configs subsection of the alertmanagers
# subsection of the alerting section
prometheus_config_alerting_alertmanagers_static_configs__custom: []

# Final options of the static_configs subsection of the alertmanagers
# subsection of the alerting section
prometheus_config_alerting_alertmanagers_static_configs: "{{
  prometheus_config_alerting_alertmanagers_static_configs__default +
  prometheus_config_alerting_alertmanagers_static_configs__custom }}"


# Default options of the first item of the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers_item__default:
  static_configs: "{{ prometheus_config_alerting_alertmanagers_static_configs }}"

# Custom options of the first item of the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers_item__custom: {}

# Final options of the first item of the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers_item: "{{
  prometheus_config_alerting_alertmanagers_item__default | combine(
  prometheus_config_alerting_alertmanagers_item__custom) }}"


# Default options of the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers__default:
  - "{{ prometheus_config_alerting_alertmanagers_item }}"

# Custom options of the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers__custom: []

# Final options of the alertmanagers subsection of the alerting section
prometheus_config_alerting_alertmanagers: "{{
  prometheus_config_alerting_alertmanagers__default +
  prometheus_config_alerting_alertmanagers__custom }}"


# Default options of the alerting section
prometheus_config_alerting__default:
  alertmanagers: "{{ prometheus_config_alerting_alertmanagers }}"

# Custom options of the alerting section
prometheus_config_alerting__custom: {}

# Final options of the alerting section
prometheus_config_alerting: "{{
  prometheus_config_alerting__default | combine(
  prometheus_config_alerting__custom) }}"


# Default options of the remote_read section
prometheus_config_remote_read__default: []

# Custom options of the remote_read section
prometheus_config_remote_read__custom: []

# Final options of the remote_read section
prometheus_config_remote_read: "{{
  prometheus_config_remote_read__default +
  prometheus_config_remote_read__custom }}"


# Default options of the remote_write section
prometheus_config_remote_write__default: []

# Custom options of the remote_write section
prometheus_config_remote_write__custom: []

# Final options of the remote_write section
prometheus_config_remote_write: "{{
  prometheus_config_remote_write__default +
  prometheus_config_remote_write__custom }}"


# Default options of the rule_files section
prometheus_config_rule_files__default: []

# Custom options of the rule_files section
prometheus_config_rule_files__custom: []

# Final options of the rule_files section
prometheus_config_rule_files: "{{
  prometheus_config_rule_files__default +
  prometheus_config_rule_files__custom }}"


# Values of the options of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs_targets_host: localhost
prometheus_config_scrape_configs_static_configs_targets_port: 9090

# Default options of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs_targets__default:
  - "{{
        prometheus_config_scrape_configs_static_configs_targets_host ~
        ':' ~
        prometheus_config_scrape_configs_static_configs_targets_port }}"

# Custom options of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs_targets__custom: []

# Final options of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs_targets: "{{
  prometheus_config_scrape_configs_static_configs_targets__default +
  prometheus_config_scrape_configs_static_configs_targets__custom }}"


# Default item of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs_item__default:
  targets: "{{ prometheus_config_scrape_configs_static_configs_targets }}"

# Custom item of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs_item__custom: {}

# Final item of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs_item: "{{
  prometheus_config_scrape_configs_static_configs_item__default | combine(
  prometheus_config_scrape_configs_static_configs_item__custom) }}"


# Default options of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs__default:
  - "{{ prometheus_config_scrape_configs_static_configs_item }}"

# Custom options of the static_configs subsection of the scrape_configs subsection
prometheus_config_scrape_configs_static_configs__custom: []


# Values of the option of the the scrape_configs section
prometheus_config_scrape_configs_job_name: prometheus
prometheus_config_scrape_configs_static_configs: "{{
  prometheus_config_scrape_configs_static_configs__default +
  prometheus_config_scrape_configs_static_configs__custom }}"

# Default item of the scrape_configs section
prometheus_config_scrape_configs_item__default:
  job_name: "{{ prometheus_config_scrape_configs_job_name }}"
  static_configs: "{{ prometheus_config_scrape_configs_static_configs }}"

# Custom item of the scrape_configs section
prometheus_config_scrape_configs_item__custom: {}

# Final item of the scrape_configs section
prometheus_config_scrape_configs_item: "{{
  prometheus_config_scrape_configs_item__default | combine(
  prometheus_config_scrape_configs_item__custom) }}"


# Default options of the scrape_configs section
prometheus_config_scrape_configs__default:
  - "{{ prometheus_config_scrape_configs_item }}"

# Custom options of the scrape_configs section
prometheus_config_scrape_configs__custom: []

# Final options of the scrape_configs section
prometheus_config_scrape_configs: "{{
  prometheus_config_scrape_configs__default +
  prometheus_config_scrape_configs__custom }}"

# Default config
prometheus_config__default:
  global: "{{ prometheus_config_global }}"
  alerting: "{{ prometheus_config_alerting }}"
  remote_read: "{{ prometheus_config_remote_read }}"
  remote_write: "{{ prometheus_config_remote_write }}"
  rule_files: "{{ prometheus_config_rule_files }}"
  scrape_configs: "{{ prometheus_config_scrape_configs }}"

# Custom config
prometheus_config__custom: {}

# Final config
prometheus_config: "{{
  prometheus_config__default | combine(
  prometheus_config__custom) }}"


# Location where to store the rules files
prometheus_rule_files_dir: /etc/prometheus/rules

# Prometheus rules (see README.md for examples)
prometheus_rule_files: {}
```


Dependencies
------------

- [`config_encoder_filters`](https://github.com/jtyr/ansible-config_encoder_filters)
- [`alertmanager`](https://github.com/jtyr/ansible-alertmanager) (optional)
- [`grafana`](https://github.com/jtyr/ansible-grafana) (optional)
- [`telegraf`](https://github.com/jtyr/ansible-telegraf) (optional)


License
-------

MIT


Author
------

Jiri Tyr
