
# Introduction
In this documentation you'll find how to install and manage Unity-IdM instance through dedicated Ansible scripts.
The concept is to have configuration of Unity-IdM externalized in a dedicated directories, and utilize those during the installation. In practice this means that software is downloaded, staged, and externalized configuration applied during installation. 

Note that externalized configuration is placed on Ansible management host, which can be different from the actual Unity-IdM instance.

# Prerequisites
Ansible scripts with 2.9 version.

Note that the Ansible scripts handles operations on Unity-IdM instance only. The database of your choice selected for Unity-IdM needs to be installed, configured and available upfront. 

## Ansible requirements
In case PostgreSQL database is used for the Unity-IdM installation, the following Python requirements are needed on the host on which a database backup is performed:
* psycopg2

When MySQL is used, then the following requirements are needed on the host on which a database backup is performed:
* MySQLdb (Python 2.x)
* PyMySQL (Python 2.7 and Python 3.X), or
* mysql (command line binary)
* mysqldump (command line binary)

# Ansible deliverables

The `devops-tooling` directory contains the Ansible playbooks, which realizes the following operations:
* Unity-IdM installation in given version,
* database backup and restore,
* start and stop the Unity-IdM instance
* check status of instance, whether it is running or not

The aforementioned directory contains also example of Ansible configuration files stored in `devops-tooling/cfg-sample`.


# Installation and configuration of devops tooling
Download the `devops-tooling` directory into your Ansible management host. For the sake of the simplicity, let's assume that installation of devops tooling was done in `$HOME/unity-devops` directory. Next, create the inventory and group variables files in the aforementioned directory. You can base on the example files that are delivered with the Ansible tooling.

With regards to the inventory file, depending upon whether you are going to install Unity-IdM on local machine, or on remote one, there are two different example files prepared. If you intend to install it on a local machine, then use inventory.local file in the example below, otherwise use inventory.remote to have installation performed on machine different than Ansible management host.

```
> cd $HOME/unity-devops
> cp devops-tooling/cfg-sample/inventory.local inventory  # or inventory.remote
> mkdir -p group_vars
> cp devops-tooling/cfg-sample/group_vars/all.yml group_vars/
```

Here is the intended directories organization, once steps in this section are completed:
```
$HOME/unity-devops   - root directory with devops tooling
  |_ devops-tooling  - downloaded Unity-IdM Ansible tooling
  |_ inventory       - Ansible inventory file
  |_ group_vars
       |_ all.yml    - Ansible group variable file
```

## Inventory
Ansible can work against multiple machines in your infrastructure. It does this by selecting systems listed in Ansible’s inventory file. Make appropriate changes to the copied example of inventory file.

## Group vars
The following table provides description of all configuration options.

| Property name | Type | Description |
| --- | --- | --- |
| `installDir` | filesystem path on Unity-IdM host | Directory name where Unity-IdM instance will be installed e.g. `{{inventory_dir}}/service`
| `unityVersion` | string | Version of Unity-IdM software that will be installed e.g. `3.8.2` |
| `unity.connectivityCheckHost` | string | The hostname or IP address with port of the Unity-IdM e.g. `localhost:2443`. This is used by Ansible scripts to query the Unity-IdM via HTTP to verify if the instance is up and running.
| `unity.confDir` | filesystem path on Ansible management host | Content of this directory will be deployed to the installed Unity-IdM under `conf` directory. This configuration option is optional.
| `unity.webContentsDir` | filesystem path on Ansible management host | Content of this directory will be deployed to the installed Unity-IdM under `webContents` directory. This configuration option is optional.
| `unity.extraDirs` | list of filesystem paths on Ansible management host | Content of pointed out directories will be deployed to the installed Unity-IdM instance to directories with names taken from paths' basename. This configuration option is optional.
| `backup.dbDumps` | filesystem path on Ansible management host | Place on the Ansible management host where database bacups are stored.
| `backup.db.backupMethod` | string | Defines a method of performing database backup (and restore). <br> Possible values:  <ul><li>`mysql-native-cmds` - operations are perform using native MySQL commands like `mysqldump` & `mysql`</li> <li>`ansible-native-mysql` - the [mysql_db](https://docs.ansible.com/devops-tooling/2.9/modules/mysql_db_module.html) Ansible module is utilized for backup and restore operations</li><li>`ansible-native-postgresql` - the [postgresql_db](https://docs.ansible.com/devops-tooling/2.9/modules/postgresql_db_module.html) Ansible module is utilized for backup and restore operations</li></ul> Note: the supported backup method for instances running with MariaDB is `ansible-native-mysql`.
| `backup.db.host` | string | The hostname or IP address of a host where database instance is running.
| `backup.db.port` | integer [0-65535] | Port number on which database instance is listening on.
| `backup.db.dbName` | string | Database name used by Unity-IdM instance.
| `backup.db.username` | string | Unity-IdM database username.
| `backup.db.password` | string | Unity-IdM database password.


# Operations

This section describes in detail implemented in Ansible scripts operations to achieve various activities.
For the sake of the simplicity, let's assume that `installDir` Ansible group variable is set to `$HOME/unity-devops/service` value.
Note that all Ansible playbook executions in this section are performed on management host in the directory where the tooling was installed.

## Installation

### Prepare Unity-IdM configuration
Before running the Ansible script to do the actual installation, you can prepare the directories on the management host with externalized Unity-IdM deliverables:

* create directory with configuration files that will be deployed as `conf` directory to the Unity-IdM instance, and set the `unity.confDir` parameter in the Ansible group variable file,
* create directory with web contents that will be deployed as `webContents` directory to the Unity-IdM instance, and set the `unity.webContentsDir` parameter in the Ansible group variable file
* create extra directories with content you would like to deploy to installed instance, and set the `unity.extraDirs` parameter in the Ansible group variable file

Each of the aforementioned step is optional, but assuming that all are used, then given the following directories structure:
```
$HOME/unity-devops   - root directory with devops tooling
  |_ devops-tooling  - downloaded Unity-IdM Ansible tooling
  |_ inventory       - Ansible inventory file
  |_ group_vars
       |_ all.yml    - Ansible group variable file
  |_ unity-config
      |_ conf        - configuration files
      |_ webContents - web contents
      |_ lib-extra1  - extra directory
      |_ lib-extra2  - extra directory
```
Configure the Ansible group variable values as follows:
* set `unity.confDir` with `{{inventory_dir}}/unity-config/conf` value,
* set `unity.webContentsDir` with `{{inventory_dir}}/unity-config/webContents` value
* set `unity.extraDirs` as shown below
    ```
    extraDirs: 
        - "{{inventory_dir}}/unity-config/lib-extra1"
        - "{{inventory_dir}}/unity-config/lib-extra2"
    ```
    NOTE: after installation, the `lib-extra1` and `lib-extra2` directories will be copied to installed Unity-IdM instance next to `conf`, `webContent` etc.

### Using Ansible variables in configuration files

The Unity-IdM configuration files pointed in `unity.confDir` setting, are considered as Ansible template files. This opens up a possibility to use Ansible variables in configuration files, so once they are deployed to target instance, variables are resolved to values set up in Ansible.  

Example: let's say you would like to have the `unityServer.core.httpServer.advertisedHost` configuration value, which is located in `unityServer.conf` file, to be dynamically set after deployment to the value configured in some Ansible variable. First, you have to introduce new variables in `group_vars/all.yml` file e.g.:

```
...
httpServer:
  advertisedHost: my.domain.com
...

```

Then use this variable in the `unityServer.conf` file as follows:
```
...
unityServer.core.httpServer.advertisedHost={{httpServer.advertisedHost}}
...

```

And after installation, the  `unityServer.conf` file will look like:
```
...
unityServer.core.httpServer.advertisedHost=my.domain.com
...

```


### Unity-IdM installation

Set the Unity-IdM version of your choice in Ansible group variable file under `unityVersion` parameter, execute:
```
ansible-playbook -i inventory devops-tooling/install.yml
```

### Installation placement
The Unity-IdM is deployed on the host which was pointed out in the inventory file.
The installation is placed in `$HOME/unity-devops/service` directory, which consists of Unity-IdM software directories, a link that points to the current working instance, and a workspace with downloaded Unity-IdM packages e.g.
```
> ls $HOME/unity-devops/service
current -> unity-server-3.8.2-20220310-063712
unity-server-3.8.2-20220310-063537
unity-server-3.8.2-20220310-063712
workspace
```


## Operating the system

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

### Changing configuration
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

## Procedure to update Unity-IdM <a id='#update-procedure'></a>
Assuming that Unity-IdM has already been installed with help of devops tooling, you can update this instance to the version of your choice. This section provides steps to achieve this.

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
    * to change configuration update files stored in directory configured under `unity.confDir` or `unity.webContentsDir` Ansible group file

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
In case Unity-IdM update was not successful, you can rollback installation to the previous version. Below is the procedure to achieve this.

1. Stop current instance, execute:
    ```
    ansible-playbook -i inventory devops-tooling/stop.yml
    ```

2. Restore database from backup of your choice, execute:
    ```
    ansible-playbook -i inventory devops-tooling/restore.yml --extra-vars "backupFilePath=$HOME/unity-devops/db-dumps/db_dumps/unitydb-v3.8.2-2022-03-07_07:39:59.sql.gz"
    ```

3. Install previous version: change `unityVersion` parameter in Ansible Group variable file, and execute:
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
