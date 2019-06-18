backups_role
=========

Backup and restore strategies for Coopdevs projects.

Requirements
------------

This role uses [Restic](https://restic.net) with the help of [restic-ansible](https://github.com/paulfantom/ansible-restic)

Role Variables
--------------
```yaml
# Restic version
backups_role_restic_version: '0.9.3'

# Location of the scripts
backups_role_path: '/opt/backup'
backups_role_script_dir: "{{ backups_role_path }}/bin"

# Overridable name of the template to generate
#+ the "prepare" script, which will be embedded inside
#+ the rendered script `backups_role_script_path`
backups_role_script_prepare_template: "cron-prepare.sh.j2"

# Complete path to rendered script formed by main, prepare and upload.
backups_role_script_path:    "{{ backups_role_script_dir }}/backup.sh"

# Location of the files generated by cron jobs
backups_role_tmp_path: '/tmp/backups'

# Lists of paths to backup
backups_role_config_paths: ['/etc', '/root', '/usr/local/etc']
backups_role_assets_paths: []

# System user, its primary group, and additional ones.
#+ Who will run scripts, restic and cron jobs
#+  and will own directories and files
backups_role_user_name: 'backups'
backups_role_user_group: 'backups'
backups_role_user_groups: ''

# Postgresql internal read-only role to perform the dump
backups_role_postgresql_user_name: "{{ backups_role_user_name }}"
# Postgres internal admin role
postgresql_user: "postgres"
backups_role_db_names: [ "postgres" ]

# Restic repository name used only in case
#+ we need to address different restic repos
backups_role_restic_repo_name: {{ inventory_hostname }}

#########################################
### WARNING! Sensible variables below ###
#########################################
###  Consider placing them inside an  ###
###  ansible-vault. As an example see ###
###  defaults/secrets.yml.example     ###
#########################################

# Password for postgresql unprivileged backups user
backups_role_postgresql_user_password:

# Restic repository password
backups_role_restic_repo_password:

# Remote bucket URL in restic format
# Example for backblaze:  "b2:bucketname:path/to/repo"
# Example for local repo: "/var/backups/repo"
backups_role_restic_repo_url:

# Backblaze "application" or bucket credentials
backups_role_b2_app_key_id:
backups_role_b2_app_key:
```


Dependencies
------------

* [paulfantom.restic](https://galaxy.ansible.com/paulfantom/restic)

Usage
-----

You will need to prepare an inventory for your hosts with the variables above. For instance, in `inventory/hosts`

### Sample postgresql playbook with backups
```yaml
# playbooks/main.yml
---
- name: Install Postgresql with automatic backups
  hosts: servers
  become: yes
  roles:
    - role: geerlingguy.postgresql
    - role: coopdevs.backups_role
```

### Playbook to restore a backup to the controller
```yaml
# playbooks/restore.yml
---
- name: Restore backup locally
  hosts: servers
  connection: local
  tasks:
  - import_role:
      name: coopdevs.backups_role
      tasks_from: restore-to-controller.yml
```

### Playbook to restore a backup to the host
```yaml
# playbooks/restore-in-situ.yml
---
- name: Restore backup to the host
  hosts: servers
  tasks:
  - import_role:
      name: coopdevs.backups_role
      tasks_from: restore-to-host.yml
```

### Restore snapshot using playbook from above
```shell
ansible-playbook playbooks/restore.yml -i inventory/hosts -l servers
```

This playbook won't install any binary globally, but it still needs root power. Therefore, you may need to provide sudo password:
```shell
ansible-playbook playbooks/restore.yml -i inventory/hosts -l servers --ask-become-pass
```

By default, it creates a directory with restic and a handy wrapper and a restore of the latest snapshot. Finally, it removes securely the wrapper, as it contains credentials that we don't want them to be lying around. However, if you prefer to install the wrapper and use it manually, you can run the playbook with by the tag `install`:

```shell
ansible-playbook playbooks/restore.yml -i inventory/hosts -l servers --tags install
```


Sensible variables
------------------
Please protect at least the variables below the "sensible variables" above. To do so, use [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html) to encrypt with a passphrase the config file with these vars.


Backblaze
---------
Backblaze provides two kind of keys: account or master, and application. There's only one account key and has power over all the buckets. We can have many app keys, that can have rw access to any, one or more buckets.

We should not use account key or reuse application keys. Even if restic passwords are different and buckets are different, one server could be able to delete backups of others, or even create more buckets and feed the bill.

Therefore, we use app keys instead of account key. As per, `ansible-restic`, it just gives the credentials to restic, regardless of the type of key. This is why we can set `ansible-restic`'s `b2_account_key` with `backup-role`'s `backups_role_b2_app_key`.

What restic calls "Account key" appears at B2 web as "Master application key".


Restic
------

Restic will create a "repository" during the Ansible provisioning. This looks like a directory inside the BackBlaze bucket being the path inside the bucket the last part of `backups_role_bucket_url`, split by `:`. If you want to place it at the root, try something like `b2:mybucket:/`. More on this at [restic docs](https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html#backblaze-b2). From the outside, you will see:  
`config  data  index  keys  locks  snapshots`

And if you decrypt it, for instance, when [mounting it](https://restic.readthedocs.io/en/latest/050_restore.html#restore-using-mount):  
`hosts  ids  snapshots  tags`

However, you may probably want to restore just a particular snapshot from the repo. To do it, use [`restic restore`](https://restic.readthedocs.io/en/latest/050_restore.html#restoring-from-a-snapshot). You will need to provide it with the snapshot id you want to resotore and the target dir to unload it. You can explore snapshots doing [`restic snapshots`](https://restic.readthedocs.io/en/latest/045_working_with_repos.html#listing-all-snapshots). A particular case is to restore the last snapshot, where you can use `latest` as snapshot id.

To restore just a file from the last snapshot instead of the whole repo, you can use the `dump` subcommand: `restic dump latest myfile > /home/user/myfile`

Remember that all restic commands need to know where to communicate to and which credentials with. So you can either pass them as parameters, or export them as environment variables. For this case, we need:

```sh
export RESTIC_REPOSITORY="b2:mybucketname:/"
export RESTIC_PASSWORD="long sentence with at least 7 words"
export B2_ACCOUNT_ID="our app key id"
export B2_ACCOUNT_KEY="our app key that is very long and has numbers and uppercase letters"
```


License
-------

GPLv3
