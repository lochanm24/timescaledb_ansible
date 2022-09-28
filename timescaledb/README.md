# timescaledb

Installs [TimescaleDB](https://www.timescale.com/) on a server, with simple, basic configuration options.

As TimescaleDB is an extension to PostgreSQL, this role takes responsibility for installation and basic configuration
of PostgreSQL, as well as installing the TimescaleDB extension.

For more elaborate PostgreSQL configuration options than those directly supported by this role, feel free to make
further modifications (e.g. by applying further changes to configuration files, or by using [PostgreSQL Ansible modules](https://docs.ansible.com/ansible/latest/modules/list_of_database_modules.html#postgresql)).

## Role Parameters (required)

| Parameter | Description |
| --------- | ----------- |
| `timescaledb_postgresql_version` | The version of PostgreSQL to install. Versions `9.6` and `10` are supported. Version `11` is also supported in 'beta'. |
| `timescaledb_databases` | A list of names of databases to create (and add the TimescaleDB extension to). |
| `timescaledb_data_directory` | The path to the directory containing the database(s). Will be created if needed. |

## Role Parameters (optional)

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `timescaledb_package_version` | The (APT) package version determining which version of TimescaleDB will be installed. | `1.4.1-de~0.10.0` |
| `timescaledb_tune_package_version` | The (APT) package version determining which version of the `timescaledb-tune` tool will be installed. | `0.7.0-de~0.4.0` |
| `timescaledb_extension_version` | The version of the TimescaleDB extension to load (should normally correspond with the 'upstream' version of the installed package). | `1.4.1` |
| `timescaledb_users` | A list of dictionaries, specifying details of database users to create. Dictionary keys `name`, `password`, `db`, `priv`, and `role_attr_flags` can all be specified (see documentation for the [`postgresql_user` Ansible module](https://docs.ansible.com/ansible/latest/modules/postgresql_user_module.html#parameters) for details of how to use these keys). | no users are created |
| `timescaledb_listen_addresses` | A list of network addresses for the TimescaleDB database to listen on. (See the [PostgreSQL documentation for `listen_addresses`](https://www.postgresql.org/docs/current/runtime-config-connection.html) for details of how to format these addresses.) | the TimescaleDB database will not listen on any network addresses |
| `timescaledb_client_auth_records_block` | A block of raw client auth records that will be inserted into the [PostgreSQL `pg_hba.conf` configuration file](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html). | only default PostgreSQL client auth records will be used |
| `timescaledb_backup_directory` | The path to the directory to be used for backups. Will be created if needed. | backups will not be enabled |
| `timescaledb_backup_retention` | If backups are enabled, the number of full (weekly) backups to retain. | 4 |

## Notes

* This role will create RUOK probe configuration snippets, and associated 'healthcheck' users in the specified database(s),
for testing the health of the database(s).
* Renaming or removing databases or users is not currently supported.

## Backups

If you specify the `timescaledb_backup_directory` parameter, nightly backups of the TimescaleDB database(s) will be
enabled, using the standard [pgBackRest](https://pgbackrest.org/) tool.

If enabled, backups will run at 02:30 (local server time) every day, with a full backup every Sunday, and incremental
backups every Monday-Saturday.

(However, the first scheduled backup to take place after backups are initially enabled will always be a full backup -- pgBackRest
will ensure this, as it is obviously not possible to perform an incremental backup if no prior full backup exists.)

By default, 4 full backups (and the subsequent incremental backups based on them) will be retained; older backups will
be automatically removed. This can be adjusted via the `timescaledb_backup_retention` parameter.

To retrieve information on backups that have been performed:

``` bash
sudo -u postgres pgbackrest info
```

See the [pgBackRest User Guide](https://pgbackrest.org/user-guide.html) for other backup operations (including restoring backups).

## Compatibility

This Ansible role only supports systems that satisfy the following criteria:

* Running a recent version of a Debian-based Linux distro
* Using a 64-bit Intel (or compatible) CPU

The role has been successfully tested against Ubuntu 18.04.

## Usage in a Playbook

For example, to install TimescaleDB and PostgreSQL 10, with a database named `mydatabase` and a user named `myuser`,
which is authorised to connect from all IPv4 addresses:

``` yaml+jinja
---
- hosts:  all
  become: true
  roles:
    - role:                           timescaledb
      timescaledb_postgresql_version: 10
      timescaledb_databases:          ['mydatabase']
      timescaledb_users:
        - name:            myuser
          password:        mypassword
          db:              mydatabase
          priv:            "ALL"
          role_attr_flags: "NOSUPERUSER,NOCREATEROLE,NOCREATEDB"
      timescaledb_listen_addresses:
        - '*'
      timescaledb_client_auth_records_block: |
        host mydatabase myuser 0.0.0.0/0 md5
```