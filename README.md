# Deploying YDB cluster with Ansible

Ansible playbooks supporting the deployment of [YDB](https://ydb.tech) clusters into VM or baremetal servers.

Currently the playbooks support the following scenarious:
* the initial deployment of YDB static (storage) nodes;
* YDB database creation;
* the initial deployment of YDB dynamic (database) nodes;
* adding extra YDB dynamic nodes to the YDB cluster;
* updating cluster configuration file and TLS certificates, with automatic rolling restart.

The following scenarious are yet to be implemented (TODO):
* configuring extra storage devices within the existing YDB static nodes;
* adding extra YDB static nodes to the existing cluster;
* removing YDB dynamic nodes from the existing cluster.

Current limitations:
* supported python interpreter version on managed should must be >= 3.7
* configuration file customization depends on the support of automatic actor system threads management, which requires YDB version 23.1.26.hotfix1 or later;
* the cluster configuration file has to be manually created;
* there are no examples for configuring the storage nodes with different disk layouts (it seems to be doable by defining different `ydb_disks` values for different host groups).

Playbooks were specifically tested on the following Linux flavours:
* Ubuntu 22.04 LTS
* AlmaLinux 8
* AlmaLinux 9
* AstraLinux Special Edition 1.7
* REDOS 7.3

# Ansible Collection - ydb_platform.ydb

Documentation for the collection.

## Playbook configuration settings

Default configuration settings are defined in the `group_vars/all` file as a set of Ansible variables. An example file is provided. Different playbook executions may require different variable values, which can be accomplished by specifying extra JSON-format files and passing those files [in the command line](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#vars-from-a-json-or-yaml-file).

The meaning and format of the variables used is specified in the table below.

| Variable | Meaning |
| --------- | ------- |
| `ydb_libidn_archive` | Enable the installation of custom-built libidn for RHEL, AlmaLinux or Rocky Linux. |
| `ydb_libidn_archive_unpack_options` | Extra flags to be passed to `tar` for unpacking custom-built libidn package. Default value: `['--strip-component=1']` |
| `ydb_archive` |  YDB server binary package in .tar.gz format |
| `ydb_archive_unpack_options` | Extra flags to be passed to `tar` for unpacking the YDB server binaries. Default value: `['--strip-component=1']` |
| `ydb_config` | The name of the cluster configuration file within the `files` subdirectory (**without** the `actor_system_config` snippet!) |
| `ydb_tls_dir` | Path to the local directory with the TLS certificates and keys, as generated by the [sample script](https://github.com/ydb-platform/ydb/tree/main/ydb/deploy/tls_cert_gen), or following the filename convention used by the sample script |
| `ydb_domain` | The name of the root domain hosting the databases, value `Root` is used in the YDB documentation |
| `ydb_disks` | Disk layout of storage nodes, defined as `ydbd_static` in the hosts file. Defined as list of structures having the following fields:<br/> `name` - physical device name (like `/dev/sdb` or `/dev/vdb`);<br/> `label` - the desired YDB data partition label, as used in the cluster configuration file (like `ydb_disk_1`) |
| `ydb_dynnodes` | Set of dynamic nodes to be ran on each host listed as `ydbd_dynamic` in the hosts file. Defined as list of structures having the following fields:<br/> `dbname` - name of the YDB database handled by the corresponding dynamic node;<br/> `instance` - dynamic node service instance name, allowing to distinguish between multiple dynamic nodes for the same database running in the same host;<br/> `offset` - integer number `0-N`, used as the offset for the standard network port numbers (`0` means using the standard ports). |
| `ydb_brokers` | List of host names running the YDB static nodes, exactly 3 (three) host names must be specified |
| `ydb_cores_static` | Number of cores to be used by thread pools of the static nodes |
| `ydb_cores_dynamic` | Number of cores to be used by thread pools of the dynamic nodes |
| `ydb_dbname` | Database name, for database creation, dynamic nodes deployment and dynamic nodes rolling restart |
| `ydb_pool_kind` | YDB default storage pool kind, as specified in the static nodes configuration file in the `storage_pool_types.kind` field |
| `ydb_database_groups` | Initial number of storage groups in the newly created database |
| `ydb_dynnode_restart_sleep_seconds` | Number of seconds to sleep after startup of each dynamic node during the rolling restart. |

## Installing the YDB cluster using Ansible

Overall installation is performed according to the [official instruction](https://ydb.tech/en/docs/deploy/manual/deploy-ydb-on-premises), with several steps automated with Ansible. The steps below are adopted for the Ansible-based process:
1. [Review the system requirements](https://ydb.tech/en/docs/deploy/manual/deploy-ydb-on-premises#requirements), and prepare the YDB hosts. Ensure that SSH access and sudo-based root privileges are available.
1. [Prepare the TLS certificates](https://ydb.tech/en/docs/deploy/manual/deploy-ydb-on-premises#tls-certificates), the provided [sample script](https://github.com/ydb-platform/ydb/tree/main/ydb/deploy/tls_cert_gen) may be used for automation of this step.
1. Download the [YDB server distribution](https://ydb.tech/en/docs/downloads/#ydb-server). It is better to use the latest binary version available.
1. Ensure that you have Python 3.8 or later installed.
1. Install the latest version of Ansible which is available for your Linux system.
1. Install the YDB Ansible collection from Github:
    ```bash
    ansible-galaxy collection install git+https://github.com/ydb-platform/ydb-ansible.git,refactor-use-collections
    ```
    Alternatively, download the current release of YDB Ansible collection from the [Releases page](https://github.com/ydb-platform/ydb-ansible/releases), and install the collection from the archive:
    ```bash
    ansible-galaxy collection install ydb-ansible-X.Y.tar.gz
    ```
1. In the new subdirectory, create the `ansible.cfg` file using the [provided example](examples/common/ansible.cfg).
1. Create the `inventory` directory, and the `inventory/50-inventory.yaml`, `inventory/99-inventory-vault.yaml` files. These files contain the host list, installation configuration and secrets to be used. The example files are provided: [inventory.yaml](examples/common/inventory/50-inventory.yaml), [inventory-vault.yaml](examples/common/inventory/99-inventory-vault.yaml).
1. Create the [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/vault_managing_passwords.html) password file as `ansible_vault_password_file`, with the password to protect the sensible secrets.
1. Encrypt `inventory/99-inventory-vault.yaml` with `ansible-vault encrypt inventory/99-inventory-vault.yaml` command. To edit this file use command `ansible-vault edit inventory/99-inventory-vault.yaml`.
1. Prepare the cluster configuration file [according to the instructions in the documentation](https://ydb.tech/en/docs/deploy/manual/deploy-ydb-on-premises#config), and save it to the `files` subdirectory. Omit the `actor_system_config` section - it will be added automatically.
1. Deploy the static nodes and initialize the cluster by running the `run-install-static.sh` script. Ensure that the playbook has completed successfully, diagnose and fix execution errors if they happen.
1. Create at least one database [according to the documentation](https://ydb.tech/en/docs/deploy/manual/deploy-ydb-on-premises#create-db). Multiple databases may run on the single cluster, each requiring the YDB dynamic node services to handle the requests. To create the database using the Ansible playbook, use the `run-create-database.sh` script. Use the `ydb_dbname` and `ydb_database_groups` variables to configure the desired database name and the initial number of storage groups in the new database.
1. Deploy the dynamic nodes running the `run-install-dynamic.sh` script. Ensure that the playbook has completed successfully, diagnose and fix execution errors if they happen.
1. Repeat steps 10-11 as necessary to create more databases, or step 11 to deploy more YDB dynamic nodes.

## Updating the cluster configuration files

To update the YDB cluster configuration files (`ydbd-config.yaml`, TLS certificates and keys) using the Ansible playbook, the following actions are necessary:
1. Ensure that the `hosts` file contains the current list of YDB cluster nodes, both static and dynamic.
1. Ensure that the configuration variable `ydbd_config` in the `group_vars/all` file points to the desired YDB server configuration file.
1. Ensure that the configuration variable `ydbd_tls_dir` points to the directory containing the desired TLS key and certificate files for all the nodes within the YDB cluster.
1. Apply the updated configuration to the cluster by running the `run-update-config.sh` script. Ensure that the playbook has completed successfully, diagnose and fix execution errors if they happen.

Notes:
1. Please take into account that rolling restart is performed node by node, and for a large cluster the process may consume a significant amount of time.
1. For Certificate Authority (CA) certificate rotation, at least two separate configuration updates are needed:
    * first to deploy the ca.crt file, containing both new and old CA certificates;
    * second to deploy the fresh server keys and certificates signed by the new CA certificate.

## What is actually done by the playbooks?

### Actions executed for installing YDB nodes
1. libaio or libaio1 is installed, depending on the operating system
1. chrony is installed and enabled to ensure time synchronization
1. jq is installed to support some scripting logic used in the playbooks
1. YDB user group and user is created
1. YDB installation directory is created
1. YDB server software binary package is unpacked into the YDB installation directory
1. YDB client package automatic update checks are disabled for the YDB user, to avoid extra messages from client commands.
1. YDB TLS certificates and keys are copied to each server
1. YDB cluster configuration file is copied to each server
1. [Transparent huge pages](https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html) (THP) are enabled on each server, which is implemented by the creation, activation and start of the corresponding systemd service.

### Actions executed for the initial deployment of YDB storage cluster
1. Installation actions are executed.
1. For each disk configured, it is checked for the existing YDB data. If none found, disk is completely re-partitioned, and obliterated. For the existing YDB data, no changes are made.
   > WARNING: the safety checks do not work for YDB disks using non-default encryption keys. DATA LOSS IS POSSIBLE if the encryption is actually used. Probably an enhancement is needed to support the encryption key to be specified in the deployment option.
1. `ydbd-storage.service` is created and configured as the systemd service.
1. `ydbd-storage.service` is started, and the playbook waits for static nodes to come up.
1. YDB blobstorage configuration is applied with the `ydbd admin blobstorage init` command.
1. The playbook waits for completion of YDB storage initialization.
1. The initial password for the `root` user is configured according to contents of the `files/secret` file.

### Actions executed for YDB dynamic nodes deployment
1. Installation actions are executed.
1. For each database configured, the list of YDB dynnode systemd services are created and configured.
1. YDB dynnode services are started.

### Actions executed for the configuration update
1. YDB TLS certificates and keys are copied to each server.
1. YDB cluster configuration file is copied to each server.
1. Rolling restart is performed for YDB storage nodes, node by node, checking for the YDB storage cluster to become healthy after the restart of each node.
1. Rolling restart is performed for YDB database nodes, server by server, restarting all nodes sitting in the single server at a time, and waiting for the specified number of seconds after each server's nodes restart.
