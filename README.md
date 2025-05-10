# 1Password Connect

Ansible role that installs a 1Password Connect server.

It uses podman, systemd quadlets, and caddy, as well as `ansible-vault` for
storing the 1Password token.

## Requirements

`ansible-vault` is required for this to work. To create a vault:

```bash
# the file must be called vars/vault.txt!
ansible-vault create vars/vault.txt
# you will be prompted for a password
```

Then, a text editor will open, which you can populate with the following
variable:

```yaml
---

# your 1password_credentials.json file from 1password's developer settings page
# goes here:
vault_onepassword_connect_credentials_json: |
  {"verifier":{"salt":"....
```

Save the file and quit the text editor to proceed.

For convenience when running Ansible playbooks, you may want to create a file in
the root of your Ansible playbook directory that contains the vault's decryption
passphrase:

```bash
echo -n 'your_vault_passphrase' > .vault-passphrase

# for a bit of extra good measure
chmod 0600 .vault-passphrase
```

You can then run Ansible with the passphrase:

```bash
ansible-playbook --vault-password-file .vault-passphrase
```

## Role Variables

Please see `defaults/main.yml` for variables that can all be set on a per-host
basis.

## Dependencies

None at the moment.

## Usage

First, create a `requirements.yml` in your Ansible setup if you haven't already,
here's an example:

```yaml
---
collections:
  - name: community.general
  - name: ansible.posix
  - name: community.crypto

roles:
  - name: charles-m-knox.onepassword-connect
    src: https://github.com/charles-m-knox/ansible-role-onepassword-connect.git
    scm: git
    version: main
```

Next, install it:

```bash
# include the -f flag to forceably re-install and get the latest (equivalent to
# upgrading)
ansible-galaxy role install -f -r requirements.yml
```

Now, in one of your host's vars' yml configuration files, define a
`onepassword_connect` object:

```yaml
onepassword_connect:
  # this directory will store the 1password-credentials.json file:
  service_dir: "{{ ansible_env.HOME }}/services/onepassword-connect"

  # the file will have 0600 permissions
  credentials_file_name: "1password-credentials.json"

  volume_name: "onepassword_connect_data"
  network_name: "onepassword_connect_network"

  # this is the api container
  connect_container:
    name: "onepassword-connect-api"
    image: "docker.io/1password/connect-api:latest"
    published_ports:
      - "127.0.0.1:8080:8080"

  # this is the sync container
  sync_container:
    name: "onepassword-connect-sync"
    image: "docker.io/1password/connect-sync:latest"
    published_ports:
      - "127.0.0.1:8081:8081"

  # this will be a drop-in caddyfile. Note that this assumes that caddy is
  # running as a rootful, non-containerized systemd service on the host.
  caddyfile:
    dest: "/etc/caddy/Caddyfile.d/onepassword_connect.caddyfile"
    become: true
    mode: "0644"
    content: |
      onepassword-connect.example.com {
        reverse_proxy 127.0.0.1:8080
      }
```

Now, in your `site.yml` (or whatever your playbook is named), use the role -
note that root access is required for some of the steps:

```yaml
- name: onepassword-connect
  hosts: all
  roles:
    - charles-m-knox.onepassword-connect
```

```bash
ansible-playbook site.yml --tags onepassword-connect --step
```

## Appendix

### How to retrieve a 1Password secret

First, update your `requirements.yml`:

```diff
---
collections:
  - name: community.general
  - name: ansible.posix
  - name: community.crypto
+  - name: onepassword.connect
```

Then install it using the command shown above:

```bash
ansible-galaxy role install -f -r requirements.yml
```

This is a test snippet that shows how to retrieve a secret from 1password:

```yaml
---

- name: Include vault
  ansible.builtin.include_vars:
    file: vars/vault.txt
  tags:
    - testing-onepassword

- name: Test onepassword connect
  onepassword.connect.field_info:
    token: "{{ connect_token }}"  # this is set in your vault
    hostname: "{{ OP_CONNECT_HOST }}"  # this can be an env var that starts with https://
    vault: "{{ OP_VAULT_ID }}"  # right-click the vault to copy its uuid
    field: "private key"
    item: "some-ssh-key-called-this"
  delegate_to: "localhost"  # just to avoid running it on different systems
  tags:
    - testing-onepassword
  register: onepassword_connect_test

- name: Show the value
  ansible.builtin.debug:
    msg: "{{ onepassword_connect_test }}"
  tags:
    - testing-onepassword
```

Finally, run it - your setup may or may not have extra vars in an env file:

```bash
ansible-playbook site.yml \
  --vault-password-file .vault-passphrase \
  --extra-vars '@.env.yml' \
  -vv \
  --tags testing-onepasword \
  -l some_hostname
```

## License

MIT

## Author Information

<https://charlesmknox.com>
