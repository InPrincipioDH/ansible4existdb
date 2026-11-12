# Setting up BSPP exist-db with ansible

## Concept of this playbook suite

This is a collection of playbooks to automate and document steps and configurations for setup of the current project infrastructure.
The scripts show sample installations. Their purpose is to explain details and to simplify experimental setups. They are not meant for production use.

The playbooks are sorted into different categories:

- `tasks`: playbooks for basic project installations
- `checklist`: tests to assess setup success (basic tasks only)
- `tasks/advanced`: playbooks for more advanced tasks. Those tasks are less straightforward. Their prerequisites are explained in the task files.
- project root: `install_everything_...` playbooks as wrapper playbooks to run everything at once. Please only use if you are familiar with the individual playbooks.

## Prerequisites

- [control node with ansible installed](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html#control-node-requirements).
- Remote VM to install exist-db to. The scripts are targeting debian 13, for other systems changes might be needed
- Registered SSH key on the VM (otherwise provide an SSH password)
- Root access to the VM
- VM is known host for the control node

## Setting up ansible project

- Provide your custom server configuration in `inventory.yml`

```bash
cp inventory.yml.example inventory.yml
```

- Create vault file from example

```bash
cp vault.yml.example vault.yml
```

- Configure your variables for backup restoration and https certificate in the vault. If you do not intend either, you may skip this step
- Encrypt the vault file (optional)

```bash
ansible-vault encrypt vault.yml
```

You may change the file contents of the vault later with

```bash
ansible-vault edit vault.yml
```

When using encrypted vault files, the decryption key must be provided on playbook runs, for example interactively with `--ask-vault-pass`.
In case of multiple vault files, all vaults must have the same password for this parameter to work.

## Usage: Service Installation and Configuration

### Install exist-db and (optionally) restore project data and settings

```bash
ansible-playbook -i inventory.yml tasks/existdb.yml --ask-vault-pass
```

After this step exist-db is reachable under the specified port. 
To access it remotely, you might have to open ports / change firewall settings in your environment.

### Set up reverse proxy and get certificate

#### Choose proxy: nginx or traefik

To set up a reverse proxy on the default ports, you may use the following command (pick either the traefik or nginx version).

```bash
ansible-playbook -i inventory.yml tasks/https-proxy_{traefik|nginx}.yml --ask-vault-pass
```

If only installing exist-db with project data, the choice of proxy does not matter.
If you want to also install the TEI publisher app, traefik is recommended due to limitations of nginx.

To set up the reverse proxy with nginx but not issue a certificate, you may skip the step with `--skip-tag optional`

To set up an https certificate with letsencrypt in the preconfigured way (certbot for nginx / http 01 challenge for traefik), port 80 of the VM needs to be open.
If this is not wanted, you need to replace the preconfigured certificate with a different certificate step (for example self signed certificate)

After this step exist-db is reachable under default port 443 (with certificate) or default port 80 (without certificate)

Alternatively you may set up a reverse proxy setup of your liking. To replicate the necessary settings for exist-db check the nginx or traefik configs in the `templates` folder or the [official documentation](https://exist-db.org/exist/apps/doc/production_web_proxying).

## Usage: Checklist for integrity of configured services

The folder `checklist` provides tasks to run automated checks on the setup of the exist-db after restoration of the project specific backup.

```bash
ansible-playbook -i inventory.yml checklist/acceptance-test.yml --ask-vault-pass
```

This command will generate `checklist/acceptance-report.md` that collects ✅ PASS and ❌ FAIL results.

The acceptance tests assumes default proxy settings and a restored backup. Changes might result in failures.
Most tests are run locally on the existdb host, but some check reachability from outside. Proxy related tests can be skipped with `--skip-tag optional`

The folder additionally contains `recommended-manual-tests.md` with some suggestions for manual checks.

## Usage: Advanced tasks

The advanced playbooks can be used to install the tei publisher app (with or without SSO) and to set up a reverse proxy location for access.
Successful setup has been tested with these tasks. However, they have various prerequisites such as
- App installation: local app package for the TEI publisher app
- App installation: npm package xst installed on the ansible control node
- SSO: preinstalled and configured keycloak instance (for SSO)

It is recommended to read and understand the playbooks before trying to use them.

Additional vars are needed by the advanced scripts (see `vars_advanced.yml`and `vault_advanced.yml.example`)

To install the tei publisher app without SSO functionality, set the `saml_enabled` var to false and use a `-nosaml` build of the app.
Be aware that this might lead to project data being publicly accessible without authentication.

## Misc

### Reverse proxy setup

Reverse proxy setup for the TEI publisher app is relatively complicated due to the underlying TEI publisher implementation.
Some pointers:
- Relevant headers for exist-db proxying: see https://exist-db.org/exist/apps/doc/production_web_proxying
- Relevant header for TEI publisher app proxying: `X-Forwarded-Host` (because it affects how `teipublisher.context.path` is handled)
- System variable `teipublisher.context.path` can be utilized to serve app under a location that is not the host basepath.
  - There is no known solution to utilize such a context path with nginx, so setting this variable with nginx is not advised
  - Once this system variable is set, the native application will become unusable. It will only be available via reverse proxy.
- Traefik allows to use an arbitrary app location. There is no official documentation for traefik on the TEI publisher side and this setup has not been used in production, so caution is advised.

### TEI Publisher 10 App

Since TEI Publisher 10, Jinks as application manager is part of it's ecosystem and it needs to be installed as a dependency.
For unknown reason, there are rare installation failures where jinks is available as an app, but its exposed modules (cpy and path) are not.
In this case jinks needs to be reinstalled, for example by removing it manually via exist-db package manager and running the ansible task again.
There is a check in place in the ansible playbook to detect this issue.