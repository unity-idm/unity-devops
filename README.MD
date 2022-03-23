
# Introduction
In this documentation you'll find how to install and manage Unity-IdM instance throught dedicated Ansible scripts.
The concept is to have configuration of Unity-IdM externalized in a dedicated directories, and utilzie those during the installation. In practice this means that software is downloaded, staged, and externalized configuration applied during installation. 

Note that externalized configuration is placed on Ansible management host, which can be different from the actual Unity-IdM instance.

# Prerequisites
Ansible scripts with 2.9 version.

Note that the Ansible scripts handles operations on Unity-IdM instance only. The database of your choice selected for Unity-IdM needs to be installed, configured and available upfront. 

# Ansible deliverables

The `devops-tooling` directory contains the Ansibe playbooks, which realizes the following operations:
* Unity-IdM installation in given version,
* databse backup and restore,
* start and stop the Unity-IdM instance
* check status of instance, whether it is running or not

The aforementioned directory contains also example of Ansible configuration files stored in `devops-tooling/cfg-sample`.

# Ansible dependencies
TBD

# Installation and configuration of devops tooling
Download the `devops-tooling` directory into your Ansible management host. For the sake of the simplicity, let's assume that installation of devops tooling was done in `$HOME/unity-devops` directory. Next, create the inventory and group variables files in the aforementioned directory. You can base on the example files that are delivered with the Ansible tooling.

With regards to the inventory file, depending upon whether you are going to install Unity-IdM on local machine, or on remote one, there are two different example files prepared. If you intend to install it on a local machine, then use inventory.local file in the example below, otherwise use inventory.remote to have installation performed on machine different than Ansible management host.

```
> cd $HOME/unity-devops
> cp devops-tooling/cfg-sample/inventory.local inventory  # or inventory.remote
> mkdir -p group_vars
> cp devops-tooling/cfg-sample/group_vars/all.yml group_vars/
```

The following files should be copied to the current directory:
* `inventory`
* `group_vars/all.yml`

## Inventory
Ansible can work against multiple machines in your infrastructure. It does this by selecting systems listed in Ansible’s inventory file. Make appropriate changes to the copied example of inventory file.

## Group vars
The following table provides description of all configuration options.

| Property name | Type | Description |
| --- | --- | --- |
| `installDir` | filesystem path on Unity-IdM host | Directory name where Unity-IdM instance will be installed e.g. `{{inventory_dir}}/service`
| `unityVersion` | string | Version of Unity-IdM software that will be installed e.g. `3.8.2` |
| `unity.connectivityCheckHost` | string | The hostname or IP address with port of the Unity-IdM e.g. `localhost:2443`. This is be used by Ansible scripts to query the Unity-IdM via HTTP to verify if the instance is up and running.
| `unity.confDir` | filesystem path on Ansible management host | Content of this directory will be deployed to the installed Unity-IdM under `conf` directory.
| `unity.webContentsDir` | filesystem path on Ansible management host | Content of this directory will be deployed to the installed Unity-IdM under `webContents` directory.
| `backup.dbDumps` | filesystem path on Ansible management host | Place on the Ansible management host where database bacups are stored.
| `backup.db.backupMethod` | string | Defines a method of performing database backup (and restore). <br> Possible values:  <ul><li>`mysql-native-cmds` - operations are perform using native MySQL commands like `mysqldump` & `mysql`</li> <li>`ansible-native-mysql` - the [mysql_db](https://docs.ansible.com/devops-tooling/2.9/modules/mysql_db_module.html) Ansible module is utilized for backup and restore operations</li><li>`ansible-native-postgresql` - the [postgresql_db](https://docs.ansible.com/devops-tooling/2.9/modules/postgresql_db_module.html) Ansible module is utilized for backup and restore operations</li><ul> 
| `backup.db.host` | string | The hostname or IP address of a host where database instance is running.
| `backup.db.port` | integer [0-65535] | Port number on which database instance is listening on.
| `backup.db.dbName` | string | Database name used by Unity-IdM instance.
| `backup.db.username` | string | Unity-IdM database username.
| `backup.db.password` | string | Unity-IdM database password.


# Operations

This section describes in detail implemented in Ansible scripts operations to achive varius activites.
For the sake of the simplicity, let's assume that `installDir` Ansible group variable is set to `$HOME/unity-devops/service` value.
Note that all Ansible playbook executions in this section are performed in on management host in the directory where the tooling was installed.

## Operating the system

### Installation placement
The Unity-IdM is deployed on the host which was pointed out int he inventory file.
The installation is placed in `$HOME/unity-devops/service` directory, which consists of Unity-IdM software directories, a link that points to the current working instance, and a workspace with downloaded Unity-IdM packages e.g.
```
> ls $HOME/unity-devops/service
current -> unity-server-3.8.2-20220310-063712
unity-server-3.8.2-20220310-063537
unity-server-3.8.2-20220310-063712
workspace
```

### Start instance
```
ansible-playbook -i inventory devops-tooling/start.yml
```

### Stop instance
```
ansible-playbook -i inventory devops-tooling/stop.yml
```

### Status of instance
```
ansible-playbook -i inventory devops-tooling/status.yml
```

### Changing coniguration
Unity-IdM configuration is stored in the directory configured under `unity.confDir` Ansible group variable. In case where a new configuration needs to be applied, operator must change the configuration files on Ansible management host, and then install Unity-IdM - please follow the [Procedure to update Unity-IdM](#update-procedure).


### Backup
```
ansible-playbook -i inventory devops-tooling/backup.yml
```
Backup files are store on the management host under the directory configured under `backup.dbDumps` Ansible group variable.

### Restore
```
ansible-playbook -i inventory devops-tooling/restore.yml --extra-vars "backupFilePath=$HOME/unity-devops/db-dumps/db_dumps/unitydb-v3.8.2-2022-03-07_07:39:59.sql.gz"
```
The `backupFilePath` is the full path to the database dump file.

## Installation
Set the Unity-IdM version of your choice in Ansible group variable file under `unityVersion` parameter, execute:
```
ansible-playbook -i inventory devops-tooling/install.yml
```

## Procedure to update Unity-IdM <a id='#update-procedure'></a>

1. Make a backup of current version, execute:
    ```
    ansible-playbook -i inventory devops-tooling/backup.yml
    ```

2. Stop current instance, execute:
    ```
    ansible-playbook -i inventory devops-tooling/stop.yml
    ```

3. From this point, it is possible to either install new Unity-IdM version, or update the configuration, or both:
    * to install new version: change `unityVersion` variable in Ansible group files,
    * to change configuration update files stored in directory configured under `unity.confDir` or `unity.webContentsDir` Andible group file

    Once above is done, execute:
    ```
    ansible-playbook -i inventory devops-tooling/install.yml
    ```

4. Start new instance, execute:
    ```
    ansible-playbook -i inventory devops-tooling/start.yml
    ```

5. Verify Unity-IdM logs.



## Rollback procedure

1. Stop current instance, execute:
    ```
    ansible-playbook -i inventory devops-tooling/stop.yml
    ```

2. Restore database from backup of your choice, execute:
    ```
    ansible-playbook -i inventory devops-tooling/restore.yml --extra-vars "backupFilePath=$HOME/unity-devops/db-dumps/db_dumps/unitydb-v3.8.2-2022-03-07_07:39:59.sql.gz"
    ```

3. Install prev version: change `unityVersion` parameter in Ansible Group variable file, and execute:
    ```
    ansible-playbook -i inventory devops-tooling/install.yml
    ```

4. Start new instance, execute:
    ```
    ansible-playbook -i inventory devops-tooling/start.yml
    ```

5. Verify Unity-IdM logs.

## Maintenance

Each time Unity-IdM is installed, a new directory with staged and configured instance is created. The instances are located on Unity-IdM host in directory that is configured under `installDir` property Ansible group variable. 
You have to clean up old directories manually, but must leave the current one. The current instance directory is the one which is pointed by the `current` link under `installDir` directory.
