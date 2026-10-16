# Setting up BSPP exist-db with ansible

## Prerequesites

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
- Encrypt the vault file

```bash
ansible-vault encrypt vault.yml
```

You may change the file contents of the vault later with

```bash
ansible-vault edit vault.yml
```

## Usage: Service Installation and Configuration

### Install exist-db and (optionally) restore project data and settings

```bash
ansible-playbook -i inventory.yml tasks/existdb.yml --ask-vault-pass
```

After this step exist-db is reachable under the specified port. 
To access it remotely, you might have to open ports / change firewall settings in your environment.

### Set up reverse proxy and get certificate

```bash
ansible-playbook -i inventory.yml tasks/https-proxy.yml --ask-vault-pass
```

To set up the reverse proxy but not issue a certificate, you may skip the step with `--skip-tag optional`

To set up an https certificate with certbot, port 80 of the VM needs to be open.
If this is not wanted, you need to replace the preconfigured certificate with a different certificate step (for example self signed certificate)

After this step exist-db is reachable under default port 443 (with certificate) or default port 80 (without certificate)

Alternatively you may set up a reverse proxy setup of your liking. To replicate the necessary settings for exist-db check the nginx in the `templates` folder or the [official documentation](https://exist-db.org/exist/apps/doc/production_web_proxying).

## Usage: Checklist for integrity of configured services

The folder `checklist` provides tasks to run automated checks on the setup of the exist-db after restoration of the project specific backup.

```bash
ansible-playbook -i inventory.yml checklist/acceptance-test.yml --ask-vault-pass
```

This command will generate `checklist/acceptance-report.md` that collects ✅ PASS and ❌ FAIL results.