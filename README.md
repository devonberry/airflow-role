# airflow

[![Build Status](https://travis-ci.org/infOpen/ansible-role-airflow.svg?branch=master)](https://travis-ci.org/infOpen/ansible-role-airflow)

Ansible role to manage Airflow installation and configuration

First role usage is to manage a single master instance, so I've not manage
worker side. If you want, free to do PR to add these features.

## Requirements

This role requires Ansible 2.0 or higher,
and platform requirements are listed in the metadata file.

## Testing

This role contains two tests methods :
- locally using Vagrant
- automatically with Travis

### Testing dependencies

> **Warning**
> Due to numpy pip package installation into isolated virtualenv,
> tests are (too) long to run with Docker. For development, perhap better to use
> Vagrant

Role tests are done with Docker, Tox and testinfra in temporaly container
- install [Docker](https://www.docker.com/)
- install [Tox](https://pypi.python.org/pypi/tox) into a virtualenv

### Running tests

#### Run playbook with Vagrant

- if Vagrant box not running
    $ vagrant up

- if Vagrant box running
    $ vagrant provision

#### Run playbook and tests with Docker

> **Warning**
> You must have an SSH keys into common location (~/.ssh/id_rsa) or set path in
>  environment variables. They will used to connect to container by SSH for
> Ansible deployment

- make test-all (for test over all Ansible version)
- or for only one version: TOXENV=py27-ansible20 make test-env

## Role Variables

The `airflow_defaults` structure defines the variables and their defaults.
These defaults get overlayed by `airflow`, so that playbooks can override
whichever variables they see fit.

> **Warning**
> Make sure to set a fernet_key and secret_key below to ensure a secure installation.

### Default role variables

```yaml
# Default configuration structures - the meat and potatoes of our config
## The `airflow` variable is intended to provide an easy means for playbooks to
## override `airflow_defaults`.  Playbooks can also reset values in the following
## so that they get picked up in the defaults:
##   * airflow_user
##   * airflow_home
##   * airflow_log
##   * airflow_pid

airflow: {}

## airflow user configuration
airflow_user: &airflow_user
  name: 'airflow'
  group: 'airflow'
  shell: '/bin/false'
  path: '/home/airflow'
  mode: '0700'

## Home directory config
airflow_home: &airflow_home
  path: '/opt/airflow'
  owner: "{{ airflow_user.name }}"
  group: "{{ airflow_user.group }}"
  mode: '0755'
  set_for_all_users: true  # Whether to set AIRFLOW_HOME in all user profiles except root

airflow_dags_folder: &airflow_dags_folder  "{{ airflow_home.path  ~ '/dags' }}"
airflow_plugins_folder: &airflow_plugins_folder  "{{ airflow_home.path  ~ '/plugins' }}"

## Log directory config
airflow_log: &airflow_log
  path: '/var/log/airflow'
  owner: "{{ airflow_user.name }}"
  group: "{{ airflow_user.group }}"
  mode: '0755'
  rotation:  # logrotate config
    - filename: '/etc/logrotate.d/airflow'
      log_pattern:
        - "/var/log/airflow/*.log"
      options:
        - 'rotate 12'
        - 'weekly'
        - 'compress'
        - 'delaycompress'
        - "create 640 {{ airflow_user.name }} {{ airflow_user.group }}"
        - 'postrotate'
        - 'endscript'

## Process ID file config
airflow_pid: &airflow_pid
  path: '/var/run/airflow'
  owner: "{{ airflow_user.name }}"
  group: "{{ airflow_user.group }}"
  mode: '0755'

## The rest of our airflow configuration
airflow_defaults:
  ## corresponds to airflow.cfg [celery] section
  celery:
    celery_app_name: 'airflow.executors.celery_executor'
    celeryd_concurrency: 16
    worker_log_server_port: 8793
    broker_url: 'sqla+mysql://airflow:airflow@localhost:3306/airflow'
    celery_result_backend: 'db+mysql://airflow:airflow@localhost:3306/airflow'
    flower_port: 5555
    default_queue: 'default'
  ## corresponds to airflow.cfg [core] section
  core:
    dags_folder: *airflow_dags_folder
    remote_base_log_folder: ''
    remote_log_conn_id: ''
    encrypt_s3_logs: False
    executor: 'SequentialExecutor'
    parallelism: 32
    dag_concurrency: 16
    dags_are_paused_at_creation: True
    non_pooled_task_slot_count: 128
    max_active_runs_per_dag: 16
    load_examples: False
    plugins_folder: *airflow_plugins_folder
    fernet_key: 'cryptography_not_found_storing_passwords_in_plain_text'
    donot_pickle: False
    dagbag_import_timeout: 30
  database:
    init: true     # run airflow initdb
    upgrade: true  # run airflow upgradedb
    connection: 'sqlite:///{{ airflow_home.path }}/airflow.db'
    pool_size: 5
    pool_recycle: 3600
  ## Runtime dependencies
  dependencies:
    # Native library dependencies
    native: "{{ airflow_native_crypto + airflow_native_hive }}"  # defined in native vars files
    # Python package dependencies
    python: "{{ airflow_python_packages_airflow + airflow_python_packages_hdfs + airflow_python_packages_hive }}"
    # Airflow modules to install
    modules:
      - name: 'crypto'
      - name: 'hdfs'
      - name: 'hive'
      - name: 'password'
  ## Python virtual environment configuration
  environment:
    virtualenv: "{{ airflow_home.path }}/venv"
    python_version: 'python2.7'
  ## DAG git repository config (optional), see http://docs.ansible.com/ansible/git_module.html for options
  git:
    repo: null
    local_dest: /tmp/airflow  # The local temporary directory
    version: 'HEAD'
    key_file: null
    accept_hostkey: false
    force: false
    ssh_opts: null
  ## Alias to workaround YAML limitation
  home: *airflow_home
  ## corresponds to airflow.cfg [email] section
  email:
    email_backend: 'airflow.utils.email.send_email_smtp'
  ## corresponds to airflow.cfg [smtp] section
  smtp:
    smtp_host: 'localhost'
    smtp_starttls: True
    smtp_ssl: False
    smtp_user: 'airflow'
    smtp_port: 25
    smtp_password: 'airflow'
    smtp_mail_from: 'airflow@airflow.com'
  ## corresponds to airflow.cfg [mesos] section
  mesos:
    master: 'localhost:5050'
    framework_name: 'Airflow'
    task_cpu: 1
    task_memory: 256
    checkpoint: False
    failover_timeout: 604800
    authenticate: False
    default_principal: 'admin'
    default_secret: 'admin'
  ## corresponds to airflow.cfg [operators] section
  operators:
    default_owner: 'Airflow'
  ## Alias to workaround YAML limitation
  log: *airflow_log
  ## Alias to workaround YAML limitation
  pid: *airflow_pid
  ### scheduler service configuration
  scheduler:
    sysinit: 'initd'   # 'initd', 'upstart' or null
    enabled: true
    runs: 0  # number of times for the scheduler to run before exiting, 0 = infinite
    pid_file: "{{ airflow_pid.path }}/airflow-scheduler.pid"
    log_file: "{{ airflow_log.path }}/airflow-scheduler.log"
    job_heartbeat_sec: 5
    scheduler_heartbeat_sec: 5
    statsd_on: false
    statsd_host: 'localhost'
    statsd_port: 8125
    statsd_prefix: 'airflow'
    max_threads: 2
  ## Alias to workaround YAML limitation
  user: *airflow_user
  ## webserver configuration (airflow.cfg + additional config)
  webserver:
    enabled: true
    sysinit: 'initd'   # 'initd', 'upstart' or null
    pid_file: "{{ airflow_pid.path }}/airflow-webserver.pid"
    log_file: "{{ airflow_log.path }}/airflow-webserver.log"
    debug: false
    base_url: 'http://localhost:8080'
    web_server_host: '0.0.0.0'
    web_server_port: 8080
    web_server_worker_timeout: 120
    secret_key: 'temporary_key'
    workers: 4
    worker_class: sync
    expose_config: true
    authenticate: true
    auth_backend: airflow.contrib.auth.backends.password_auth
    filter_by_owner: false

## Merge the user config on top of defaults to get our final config
airflow_config: "{{ airflow_defaults | combine(airflow, recursive=True) }}"
```

## Dependencies

None

## Example Playbook

    - hosts: servers
      roles:
         - { role: infOpen.airflow }

## License

MIT

## Author Information

Devon Berry

based on the ansible-role-airflow Ansible Galaxy role by

Alexandre Chaussier (for Infopen company)
- http://www.infopen.pro
- a.chaussier [at] infopen.pro
