=====================================
F5OS Configuration Backup and Restore
=====================================

To completely backup the VELOS system, you’ll need to backup each tenant TMOS configuration, each F5OS chassis partition configuration, as well as the F5OS system controller configuration. Tenant backup utilizes the same backup and recovery procedures as existing BIG-IP devices/guests because the tenants themselves are running TMOS. For the F5OS layer (system controllers and chassis partitions), a different backup mechanism is utilized because F5OS configuration management is based on ConfD.  

The ConfD process manages the F5OS configuration on a VELOS system. The system stores the configuration in its configuration database (CDB). There are separate ConfD databases for the system controller layer, and for each chassis partition.

At the chassis level, the F5OS configuration contains data that includes the following:

•	DNS
•	Network time protocol (NTP)
•	Authentication servers
•	Logging setup
•	High availability (HA) setup
•	Management IP addresses of the system controllers
•	Product license
•	Basic configuration of existing partitions, such as the partition name, floating management IP address, and the partition-to-slot mapping

At the chassis partition level, the F5OS configuration contains data that includes the following:

•	Portgroups of the assigned slots/nodes/blades
•	Virtual Local Area Networks (VLANs)
•	Logging setup
•	Authentication servers
•	Product license
•	HA setup

To perform a complete backup of the VELOS system, you must:

•	Back up the configuration data at the system controller and at each chassis partition.
•	Back up any deployed tenants using the tenants’ backup mechanism (i.e. a UCS).

More detail is covered in the following solution article:

https://support.f5.com/csp/article/K50135154

Backing Up the System Controller Database
=========================================

Backing Up the System Controller Database via CLI
-------------------------------------------------

You can back up the system controller configuration database using the **system database config-backup** command when in **config** mode. The file will be saved in the path of **/configs** automatically. You can then list the contents of that directory to ensure the file is there using the **file list path** command.

.. code-block:: bash

    syscon-1-active# config
    Entering configuration mode terminal
    syscon-1-active(config)# system database config-backup name chassis2-sys-controller-backup-2-26-21
    response Succeeded.
    syscon-1-active(config)# exit 

    syscon-1-active# file list path configs
    entries {
        name 
    chassis2-sys-controller-backup-2-26-21
    test-backup
    }
    syscon-1-active# 



Backing Up the System Controller Database via webUI
---------------------------------------------------

Using the system controller webUI you can backup the ConfD configuration database using the **System Settings -> Configuration Backup** page. Click the **Create** button and provide a name for the backup file.

.. image:: images/velos_f5os_configuration_backup_and_restore/image1.png
   :width: 45%

.. image:: images/velos_f5os_configuration_backup_and_restore/image2.png
   :width: 45%

Backing Up the System Controller Database via API
-------------------------------------------------

The following API call will backup the system controller.

.. code-block:: bash

    POST https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/openconfig-system:system/f5-database:database/f5-database:config-backup

In the body of the API call, supply the name of the file that you want to save. 

.. code-block:: json

    {
        "f5-database:name": "SYSTEM-CONTROLLER-DB-BACKUP{{currentdate}}"
    }


**Note: In the current F5OS releases the ConfD system database can be backed up via CLI, webUI, or API but it cannot be restored using the F5OS webUI. This may be added in a subsequent release.**

Copying System Controller Database Backup to an External Location
=================================================================

Once the database backup has been completed, you should copy the file to an external location so that the system can be restored in the case of a total failure. You can download the database configuration backup using the CLI, webUI, or API. 

Copying System Controller Database Backup to an External Location via webUI
---------------------------------------------------------------------------

In the webUI use the **System Settings -> File Utilities** page and from the dropdown select **configs** to see the previously saved backup file. Here you can **Import** or **Export**, as well as **Upload** and **Download** configuration files. Note that the Import and Export options to transfer files requires an external HTTPS server, while the Upload and Download options will move files form your local browser. 

.. image:: images/velos_f5os_configuration_backup_and_restore/image3.png
  :align: center
  :scale: 70%

.. image:: images/velos_f5os_configuration_backup_and_restore/image4.png
  :align: center
  :scale: 70%


Copying System Controller Database Backup to an External Location via CLI
-------------------------------------------------------------------------

To transfer a file using the CLI use the **file list** command to see the contents of the **configs** directory. Note the previously saved file is listed.

.. code-block:: bash

    syscon-2-active# file list path configs/
    entries {
        name 
    CONTROLLER-API-DB-BACKUP2021-08-19
    SYSTEM-CONTROLLER-DB-BACKUP2021-08-27
    controller-backup-08-17-21
    my-backup
    }


To transfer the file from the CLI you can use the **file export** command. The option below is exporting to a remote HTTPS server. there are options to trasnfer using SFTP, and SCP as well.

.. code-block:: bash

    syscon-2-active# file export local-file configs/SYSTEM-CONTROLLER-DB-BACKUP2021-08-27 remote-host 10.255.0.142 remote-file /upload/upload.php username corpuser insecure 
    Value for 'password' (<string>): ********
    result File transfer is initiated.(configs/SYSTEM-CONTROLLER-DB-BACKUP2021-08-27)
    syscon-2-active#

To check on status of the export use the **file transfer-status** command:

.. code-block:: bash

    syscon-1-active# file transfer-status                                                                                                                                   
    result 
    S.No.|Operation  |Protocol|Local File Path                                             |Remote Host         |Remote File Path                                            |Status            
    1    |Export file|HTTPS   |configs/SYSTEM-CONTROLLER-DB-BACKUP2021-08-27                |10.255.0.142        |/upload/upload.php                                          |Completed|Fri Aug 27 19:48:41 2021
    2    |Export file|HTTPS   |/mnt/var/confd/configs/chassis1-sys-controller-backup-2-26-21|10.255.0.142        |chassis1-sys-controller-backup-2-26-21                      |Failed to open/read local data from file/application
    3    |Export file|HTTPS   |/mnt/var/confd/configs/chassis1-sys-controller-backup-2-26-21|10.255.0.142        |/backup                                                     |Failed to open/read local data from file/application

If you don’t have an external HTTPS server that allows uploads, then you can log into the system controllers floating IP address with root access and scp the file from the shell. Go to the **/var/confd/configs** directory and scp the file to an external location. Note in the CLI and webUI the path is simplified to configs, but in the underlying file system it is actually stored in the **/var/confd/configs** directory.

.. code-block:: bash

    [root@controller-2 ~]# ls /var/confd/configs/
    controller-backup-08-17-21  my-backup
    [root@controller-2 ~]# scp /var/confd/configs/controller-backup-08-17-21 root@10.255.0.142:/var/www/server/1
    The authenticity of host '10.255.0.142 (10.255.0.142)' can't be established.
    ECDSA key fingerprint is SHA256:xexN3pt/7xGgGNFO3Lr77PHO2gobj/lV6vi7ZO7lNuU.
    ECDSA key fingerprint is MD5:ff:06:0f:a8:5f:64:92:7b:42:31:aa:bf:ea:ee:e8:3b.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '10.255.0.142' (ECDSA) to the list of known hosts.
    root@10.255.0.142's password: 
    controller-backup-08-17-21                                                       100%   77KB  28.8MB/s   00:00    
    [root@controller-2 ~]# 

Copying System Controller Database Backup to an External Location via API
-------------------------------------------------------------------------

To copy a ConfD configuration backup file from the system controller to a remote https server use the following API call:

.. code-block:: bash

    POST https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/f5-utils-file-transfer:file/export

.. code-block:: json

    {
        "f5-utils-file-transfer:insecure": "",
        "f5-utils-file-transfer:protocol": "https",
        "f5-utils-file-transfer:username": "corpuser",
        "f5-utils-file-transfer:password": "Passw0rd!!",
        "f5-utils-file-transfer:remote-host": "10.255.0.142",
        "f5-utils-file-transfer:remote-file": "/upload/upload.php",
        "f5-utils-file-transfer:local-file": "configs/SYSTEM-CONTROLLER-DB-BACKUP{{currentdate}}"
    }


Backing Up Chassis Partition Databases
======================================

In addition to backing up the system controller database, you should backup the configuration database on each chassis partition within the VELOS system. In the example below there are two chassis partitions currently in use; **Production** and **Development**. Both must be backed up and archived off of the VELOS system.

Backing Up Chassis Partition Databases via CLI
----------------------------------------------

Log directly into the chassis partition Production's management IP address and enter **config** mode. Use the **system database config-backup** command to save a copy of the chassis partition config database. Then list the file using the **file list** command.

.. code-block:: bash

    Production-1# config
    Entering configuration mode terminal
    Production-1(config)# system database config-backup name chassis-partition-production-08-17-2021
    result Database backup successful.
    Production-1(config)# exit
    Production-1# file list path configs/
    entries {
        name 
    chassis-partition-production-08-17-2021
    }
    Production-1# 


Log directly into the chassis partition development's management IP address and enter **config** mode. Use the **system database config-backup** command to save a copy of the chassis partitions config database. Then list the file using the **file list** command.

.. code-block:: bash

    development-1# config
    Entering configuration mode terminal
    development-1(config)# system database config-backup name chassis-partition-development-08-17-2021
    result Database backup successful.
    development-1(config)# exit
    development-1# file list path configs/
    entries {
        name 
    chassis-partition-development-08-17-2021
    }
    development-1# 


Backing Up Chassis Partition Databases via webUI
------------------------------------------------


This can also be done from each chassis partition’s webUI interface. Log into the chassis partition webUI. Then go to **System Utilities -> Configuration Backup**. Click **Create** to save the ConfD database configuration and provide a name. 

.. image:: images/velos_f5os_configuration_backup_and_restore/image5.png
  :align: center
  :scale: 70%

Backing Up Chassis Partition Databases via API
------------------------------------------------


You’ll need to do this for each chassis partition in the system. To backup the chassis partition databases via API use the following API command:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/openconfig-system:system/f5-database:database/f5-database:config-backup


.. code-block:: json

    {
        "f5-database:name": "Production-DB-BACKUP{{currentdate}}"
    }

Repeat this for each chassis partition.

Export Backups From the Chassis Partitions
==========================================

Next copy the backup files to a location outside of VELOS. The file can be copied off via the chassis partitions CLI, webUI, or API. 

Export Backup From the Chassis Partition webUI
----------------------------------------------

You can copy the backup file out of the chassis partition using the **Systems Settings > File Utilities** menu in the webUI. Use the Base Directory drop down menu to select **configs** directory, you should see a copy of the file created there:

.. image:: images/velos_f5os_configuration_backup_and_restore/image6.png
  :align: center
  :scale: 70%

You can highlight the file and then click the **Export** button. You when then be prompted to enter the details for a remote HTTPS server so that the file can be copied out of the chassis partition:

.. image:: images/velos_f5os_configuration_backup_and_restore/image7.png
  :align: center
  :scale: 70%

If you select *Download**, then an option will appear to download through your browser to your local client machine.


Export Backup From the Chassis Partition CLI
--------------------------------------------

To transfer a file using the CLI, use the **file list** command to see the contents of the **configs** directory. Note, the previously saved file is listed. You will need to repeat this for all chassis partitions in the VELOS system.

To export the backup for the chassis partition **Production**, first list the contents of the configs directory:

.. code-block:: bash

    Production-1# file list path configs/
    entries {
        name 
    chassis-partition-Production-08-17-2021
    }
    Production-1# 

To transfer the file from the CLI, you can use the **file export** command. Note that the file export command requires either a remote HTTPS, SFPT, or SCP server that the file can be posted to. 

.. code-block:: bash

    Production-1# file export local-file configs/chassis-partition-Production-08-17-2021 remote-host 10.255.0.142 remote-file /upload/upload.php username corpuser insecure
    Value for 'password' (<string>): ********
    result File transfer is initiated.(configs/chassis-partition-Production-08-17-2021)
    Production-1#

You can use the CLI command **file transfer-status** to see if the file was copied successfully or not:

.. code-block:: bash

    Production-1# file transfer-status                                                                                                                                       
    result 
    S.No.|Operation  |Protocol|Local File Path                                             |Remote Host         |Remote File Path                                            |Status            |Time                
    1    |Export file|HTTPS   |configs/3-20-2021-Production-backup                       |10.255.0.142        |/upload/upload.php                                          |Failed to open/read local data from file/application|Fri Aug 27 20:05:34 2021
    2    |Export file|HTTPS   |configs/chassis-partition-Production-08-17-2021           |10.255.0.142        |/upload/upload.php                                          |         Completed|Fri Aug 27 20:06:22 2021

    Production-1# 


If you do not have a remote HTTPS, SCP, or SFTP server with the proper access to POST files, then you can copy the chassis partition backups from the system controller shell (Note, there is no shelll access via the chassis partition IP). You’ll need to login to the system controllers shell using the root account. Once logged in list the contents of the **/var/F5** directory. You’ll notice **partition<ID>** directories, where <ID> equals the ID assigned to each partition.

.. code-block:: bash

    [root@controller-2 ~]# ls -al /var/F5/
    total 36
    drwxr-xr-x. 10 root root 4096 Mar 10 21:43 .
    drwxr-xr-x. 40 root root 4096 Mar  3 04:17 ..
    drwxr-xr-x.  3 root root 4096 Feb  8 19:58 controller
    drwxr-xr-x.  5 root root 4096 Feb  8 19:58 diagnostics
    drwxr-xr-x.  2 root root 4096 Feb  8 19:58 fips
    drwxr-xr-x. 24 root root 4096 Mar  3 04:27 partition1
    drwxr-xr-x.  3 root root   20 Mar 10 17:54 partition2
    drwxr-xr-x. 24 root root 4096 Mar  4 15:52 partition3
    drwxr-xr-x. 22 root root 4096 Mar 10 21:45 partition4
    drwxr-xr-x.  3 root root 4096 Feb  9 16:08 sirr
    [root@controller-2 ~]# 

The backup files for each partition are stored in the **/var/F5/partition<ID>/configs** directory. You will need to copy off each chassis partition backup file. You can use SCP to do this from the shell.

.. code-block:: bash

    [root@controller-2 ~]# ls -al /var/F5/partition4/configs
    total 52
    drwxrwxr-x.  2 root admin    43 Mar 20 06:10 .
    drwxr-xr-x. 22 root root   4096 Mar 10 21:45 ..
    -rw-r--r--.  1 root root  46954 Mar 20 06:10 3-20-2021-Production-backup
    [root@controller-2 ~]# 

Below is an example using SCP to copy off the backup file from partition ID 4, you should do this for each of the partitions:

.. code-block:: bash

    [root@controller-2 ~]# scp /var/F5/partition4/configs/3-20-2021-Production-backup root@10.255.0.142:/var/www/server/1/.
    root@10.255.0.142's password: 
    3-20-2021-Production-backup                                                             100%   46KB  23.7MB/s   00:00    
    [root@controller-2 ~]# 
    
Now repeat the same steps for each chassis partition in the system. 

Export Backup From the Chassis Partition API
--------------------------------------------

Each chassis partition in the system needs to be backed up independently. Below is an API example of exportinh the backuo up the chassis partition **Development**. Note the API call is sent to the chassis partition IP address. Currently a remote HTTPS, SCP, or SFTP server is required to export the copy of the configuration backup.

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition2_ip}}:8888/api/data/f5-utils-file-transfer:file/export

.. code-block:: json

    {
        "f5-utils-file-transfer:insecure": "",
        "f5-utils-file-transfer:username": "corpuser",
        "f5-utils-file-transfer:password": "Passw0rd1!",
        "f5-utils-file-transfer:local-file": "configs/development-DB-BACKUP{{currentdate}}",
        "f5-utils-file-transfer:remote-host": "10.255.0.142",
        "f5-utils-file-transfer:remote-port": 0,
        "f5-utils-file-transfer:remote-file": "/upload/upload.php"
    }

To check on the status of the file export you can use the following API call to check the transfer-status:

.. code-block:: bash

  POST https://{{velos_chassis1_chassis_partition2_ip}}:8888/api/data/f5-utils-file-transfer:file/transfer-status

In the body of the post use the following json payload to denote the path and file name to be exported.

.. code-block:: json

    {
        "f5-utils-file-transfer:file-name": "configs/development-DB-BACKUP{{currentdate}}"
    }

A status similar to the output below will be seen.

.. code-block:: json

    {
        "f5-utils-file-transfer:output": {
            "result": "\nS.No.|Operation  |Protocol|Local File Path |Remote Host  |Remote File Path   |Status  |Time  \n1    |Export file|HTTPS   |configs/development-DB-BACKUP2021-08-27 |10.255.0.142 |/upload/upload.php | Completed|Fri Aug 27 20:18:12 2021"
        }
    }

Repeat this step for all the other chassis partitions in the system.

Backing up Tenants
==================

Backup all tenants using a UCS archive or other mechanism so that they can be restored after the system controller and chassis partitions are restored. Another alternative to UCS backup/restore of tenants is using Declarative Onboarding and AS3. If tenants are configured using DO and AS3 initially, and the declarations are saved, they can be replayed to restore a tenant. BIG-IQ could be used for this purpose as AS3 and DO declarations can be sent through BIG-IQ.

Resetting the System (Not for Production)
=========================================

For a proof-of-concept test, this section will provide steps to wipe out the entire system configuration in a graceful manner. This is not intended as a workflow for production environments, as you would not typically be deleting entire system configurations, instead you would be restoring pieces of the configuration in the case of failure. 

The first step would be to ensure you have completed the previous sections, and have created backups for the system controllers, each chassis partition, and each tenant. These backups should have been copied out of the VELOS system to a remote server so that they can be copied back into the system and used to restore after it has been reset.


Remove Partitions and Reset Controller via CLI
----------------------------------------------

The first step is to ensure each chassis partition’s ConfD database has been **reset-to-default**. This will wipe out all tenant configurations and networking as well as all the system parameters associated with each chassis partition.

For the Development chassis partition:

.. code-block:: bash

    Development-1# config
    Development-1(config)# system database reset-to-default proceed  
    Value for 'proceed' [no,yes]: yes
    result Database reset-to-default successful.
    Development-1(config)# 
    System message at 2021-03-02 22:51:54...
    Commit performed by admin via tcp using cli.
    Development-1(config)# 


For the Production chassis partition:

.. code-block:: bash

    Production-1# config 
    Entering configuration mode terminal
    Production-1(config)# system database reset-to-default proceed 
    Value for 'proceed' [no,yes]: yes
    result Database reset-to-default successful.
    Production-1(config)# 
    System message at 2021-03-02 23:01:50...
    Commit performed by admin via tcp using cli.
    Production-1(config)# 

Once the partition configurations have been cleared, you’ll need to login to the system controller CLI via the floating IP address. You’ll need to put all slots back into the **none** partition and **commit** the changes. This will allow the partitions to be deleted in the next step.

.. code-block:: bash

    syscon-2-active(config)# slots slot 1-3 partition none
    syscon-2-active(config-slot-1-3)# commit 
    Commit complete.
    syscon-2-active(config-slot-1-3)#


Then remove the partitions from the system controller. In this case we will remove the chassis partitions called **Production** and **Development**.

.. code-block:: bash

    syscon-2-active(config)# no partitions partition Production 
    syscon-2-active(config)# no partitions partition Development 
    syscon-2-active(config)# commit 
    Commit complete.
    syscon-2-active(config)# 


For the final step, reset the system controllers ConfD database. This will essentially wipe out all partitions and all of the system controller configuration essentially setting it back to factory default.


.. code-block:: bash

    syscon-2-active(config)# system database config reset-default-config true
    syscon-2-active(config)# commit

Once this has been committed, both controllers need to be rebooted manually and in quick succession of each other. Login to the active controller and enter **config** mode, and then issue the **system reboot controllers controller standby** command, this will reboot the standby controller first. Run the same command again but this time reboot the **active** controller immediately after resetting the primary controller. You don't want any sort of long pause (minutes) between the resets. Ideally these commands should be run back to back.

.. code-block:: bash

    syscon-1-active(config)# system reboot controllers controller standby

    syscon-1-active(config)# system reboot controllers controller active

The system controllers should reboot, and their configurations will be completely wiped clean. You will need to login via the console / CLI to restore out-of-band networking connectivity, and then the previously archived configurations can be copied back and restored.


Remove Partitions and Reset Controller via API
----------------------------------------------

The reset-to-default for the chassis partition database is not supported via the webUI. This can be done via an API call to the chassis partition IP address. Below is an example sending the database reset-to-default command to the chassis partition called Production:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/openconfig-system:system/f5-database:database/f5-database:reset-to-default

The body of the API call must have the following:

.. code-block:: json

    {
    "f5-database:proceed": "yes"
    }

Repeat this for the other chassis partitions in the system, in this case send an API call to the IP address of the chassis partition Development:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition2_ip}}:8888/restconf/data/openconfig-system:system/f5-database:database/f5-database:reset-to-default

The body of the API call must have the following:

.. code-block:: json

    {
    "f5-database:proceed": "yes"
    }

Next, send an API call to the system controller IP address to re-assign any slots that were previously part of a chassis partition to the partition **none**. In the example below slots 1-2 were assigned to the chassis partition Production, and slot3 was assigned to the chassis partition Development. All 3 slots will be moved to the partition none. 


.. code-block:: bash

    POST https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/

All 3 slots are assigned to partition none.

.. code-block:: json

    {
        "f5-system-slot:slots": {
            "slot": [
                {
                    "slot-num": 1,
                    "enabled": true,
                    "partition": "none"
                },
                {
                    "slot-num": 2,
                    "enabled": true,
                    "partition": "none"
                },
                {
                    "slot-num": 3,
                    "enabled": true,
                    "partition": "none"
                }
            ]
        }
    }

Once the slots have been removed from the partitions, you can Delete any chassis partitions that were configured. In this case both **Production** and **Development** chassis partitions will be deleted by sending API calls to the system controller IP address:

.. code-block:: bash

    DELETE https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/f5-system-partition:partitions/partition=Production

    DELETE https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/f5-system-partition:partitions/partition=Development

The last step in the reset procedure is to set the system controllers ConfD database back to default.

.. code-block:: bash

    POST https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/openconfig-system:system/f5-database:database/f5-database:config

.. code-block:: json

    {
    "f5-database:reset-default-config": "true"
    }

Once this has been committed, both controllers need to be rebooted manually and in quick succession of each other. Login to the active controller and enter **config** mode, and then issue the **system reboot controllers controller standby** command, this will reboot the standby controller first. Run the same command again but this time reboot the **active** controller immediately after resetting the primary controller. You don't want any sort of long pause (minutes) between the resets. Ideally these commands should be run back to back.

.. code-block:: bash





The system controllers should reboot, and their configurations will be completely wiped clean. You will need to login via the CLI to restore out-of-band networking connectivity, and then the previously archived configurations can be copied back and restored.  

Remove Partitions and Reset Controller via webUI
------------------------------------------------

In the system controller webUI go to the **Chassis Partitions** page. Select the chassis partition you wish to delete by using the check box, then click the **Delete** button. The webUI will automatically remove the slots and return them to the **none** chassis partition before deleting the selected chassis partition. You should delete all partitions except for **default**. 

.. image:: images/velos_f5os_configuration_backup_and_restore/image8.png
  :align: center
  :scale: 70%

There is no capability in the webUI currently to reset the system controller database. You’ll need to use the API or CLI to perform that function.

Restoring Out-of-Band Connectivity and Copying Archived Configs into the Controller
===================================================================================

You will need to login to the system controller console port since all the networking configuration has now been wiped clean. You will login with the default username/password of admin/admin, since any previous accounts will have been wiped clean. On first login you will be prompted to change your password. Note below that the current console is connected to the standby controller, you’ll need to connect to the console of the active controller to make further changes:

.. code-block:: bash

    controller-1 login: admin
    Password: 
    You are required to change your password immediately (root enforced)
    Changing password for admin.
    (current) UNIX password: admin
    New password: **************
    Retype new password: **************
    Last failed login: Fri Sep 10 14:49:55 UTC 2021 on ttyS0
    There was 1 failed login attempt since the last successful login.
    Last login: Thu Sep  2 14:09:57 on ttyS0
    Welcome to the F5OS System Controller Management CLI
    admin connected from 127.0.0.1 using console on syscon-1-standby
    syscon-1-standby# 

Logout of the system and login as root using the new password you just created for the admin account, you’ll be prompted to change the password again. There is a bug in the current F5OS version where the config directory is getting deleted on wiping out of the database, and it is not restored. Until that issue is resolved the recommended workaround is to create a new backup of the system controller configuration and that will create the required config directory. Note you will not restore from this backup, instead you will restore from the one taken earlier before the reset. 

.. code-block:: bash

    syscon-1-active# config
    Entering configuration mode terminal
    syscon-1-active(config)# system database config-backup name dummy-backup
    response Succeeded.
    syscon-1-active(config)# exit 

    syscon-1-active# file list path configs
    entries {
        name 
    dummy-backup
    test-backup
    }
    syscon-1-active# 



To transfer files into the system controller you’ll have to manually configure the out-of-band networking first. In the case below the system controller out-of-band ethernet ports were aggregated into a LAG before the system was reset. This needs to be recreated, and then static and floating out-of-band IP addresses are assigned as well as a prefix length and gateway.

.. code-block:: bash

    syscon-1-active# config
    syscon-1-active(config)# interfaces interface mgmt-aggr
    Value for 'config type' [a12MppSwitch,aal2,aal5,actelisMetaLOOP,...]: ieee8023adLag
    syscon-1-active(config-interface-mgmt-aggr)# config name mgmt-aggr
    syscon-1-active(config-interface-mgmt-aggr)# aggregation config lag-type LACP 
    syscon-1-active(config-interface-mgmt-aggr)# exit
    syscon-1-active(config)# lacp interfaces interface mgmt-aggr
    syscon-1-active(config-interface-mgmt-aggr)# config name mgmt-aggr
    syscon-1-active(config-interface-mgmt-aggr)# exit
    syscon-1-active(config)# interfaces interface 1/mgmt0 
    syscon-1-active(config-interface-1/mgmt0)# config name 1/mgmt0
    syscon-1-active(config-interface-1/mgmt0)# config type ethernetCsmacd 
    syscon-1-active(config-interface-1/mgmt0)# ethernet config aggregate-id mgmt-aggr 
    syscon-1-active(config-interface-1/mgmt0)# exit
    syscon-1-active(config)# exit
    yscon-1-active(config)# interfaces interface 2/mgmt0  
    syscon-1-active(config-interface-2/mgmt0)# config name 2/mgmt0
    syscon-1-active(config-interface-2/mgmt0)# config type ethernetCsmacd 
    syscon-1-active(config-interface-2/mgmt0)# ethernet config aggregate-id mgmt-aggr
    syscon-1-active(config-interface-2/mgmt0)# 
    syscon-1-active(config)# system mgmt-ip config ipv4 controller-1 address 10.255.0.145
    syscon-1-active(config)# system mgmt-ip config ipv4 controller-2 address 10.255.0.146
    syscon-1-active(config)# system mgmt-ip config ipv4 floating address 10.255.0.147
    syscon-1-active(config)# system mgmt-ip config ipv4 gateway 10.255.0.1
    syscon-1-active(config)# system mgmt-ip config ipv4 prefix-length 24
    syscon-1-active(config)# commit 
    Commit complete.


Importing System Controller Backups
===================================

Once the system is configured and out-of-band connectivity is restored, you can now copy the ConfD database archives back into the system controllers. If you are in the bash shell you can simply SCP the file into the **/var/confd/configs** directory. If it doesn’t exist, you can create it by creating a dummy backup of the system controllers configuration as outlined earlier.


Next SCP the file from a remote server:

.. code-block:: bash

    scp root@10.255.0.142:/var/www/server/1/upload/SYSTEM-CONTROLLER-DB-BACKUP2021-09-10 .


Importing System Controller Backups via CLI
-------------------------------------------

To import the file using the F5OS CLI you must have a remote HTTPS, SFTP, or SCP server to host the file. Use the **file import** command as seen below to import the file into the **configs** directory. You can then check the **file transfer-status** and list the contents of the config directory using the **file list path configs** command.

.. code-block:: bash

    syscon-1-active# file import remote-host 10.255.0.142 remote-file /upload/SYSTEM-CONTROLLER-DB-BACKUP2021-09-10 local-file configs/SYSTEM-CONTROLLER-DB-BACKUP2021-09-10 username corpuser insecure
    Value for 'password' (<string>): ********
    result File transfer is initiated.(configs/SYSTEM-CONTROLLER-DB-BACKUP2021-09-10)


    syscon-1-active# file transfer-status 
    result 
    S.No.|Operation  |Protocol|Local File Path                                             |Remote Host         |Remote File Path                                            |Status            |Time                
    1    |Import file|HTTPS   |configs/SYSTEM-CONTROLLER-DB-BACKUP2021-09-10               |10.255.0.142        |/upload/SYSTEM-CONTROLLER-DB-BACKUP2021-09-10               |         Completed|Wed Sep 15 01:57:39 2021


    syscon-1-active# file list path configs/
    entries {
        name 
    dummy-backup
    SYSTEM-CONTROLLER-DB-BACKUP2021-09-10
    }
    syscon-1-active# 

Importing System Controller Backups via API
-------------------------------------------

Post the following API call to the system controllers IP address to import the archived ConfD backup file from a remote HTTPS server to the configs directory on the system controller.

.. code-block:: bash

    POST https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/f5-utils-file-transfer:file/import

.. code-block:: json

    {
        "f5-utils-file-transfer:insecure": "",
        "f5-utils-file-transfer:protocol": "https",
        "f5-utils-file-transfer:username": "corpuser",
        "f5-utils-file-transfer:password": "Passw0rd!!",
        "f5-utils-file-transfer:remote-host": "10.255.0.142",
        "f5-utils-file-transfer:remote-file": "/upload/SYSTEM-CONTROLLER-DB-BACKUP{{currentdate}}",
        "f5-utils-file-transfer:local-file": "configs/SYSTEM-CONTROLLER-DB-BACKUP{{currentdate}}"
    }

You may query the transfer status of the file via the following API command:

.. code-block:: bash

    POST https://{{velos_chassis1_system_controller_ip}}:8888/api/data/f5-utils-file-transfer:file/transfer-status

.. code-block:: json

    {
        "f5-utils-file-transfer:file-name": "configs/SYSTEM-CONTROLLER-DB-BACKUP{{currentdate}}"
    }

If you want to list the contents of the config directory via API use the following API command:

.. code-block:: bash

    POST https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/f5-utils-file-transfer:file/list

.. code-block:: json

    {
    "f5-utils-file-transfer:path": "configs"
    }

You’ll see the contents of the directory in the API response:

.. code-block:: json

    {
        "f5-utils-file-transfer:output": {
            "entries": [
                {
                    "name": "\nSYSTEM-CONTROLLER-DB-BACKUP2021-09-10"
                }
            ]
        }
    }


Importing System Controller Backups via webUI
-------------------------------------------

You can use the **System Settings -> File Utilities** page to import or upload an archived system controller backup from a remote HTTPS, SFTP, or SCP server. Use the drop-down option for **Base Directory** and choose **configs** to see the current files in that directory, and to import or export files. Choose the **Import** option and a popup will appear asking for the details of how to obtain the remote file. The **Upload** option will allow you to upload from you client machine via the browser.

.. image:: images/velos_f5os_configuration_backup_and_restore/image9.png
  :align: center
  :scale: 70%

.. image:: images/velos_f5os_configuration_backup_and_restore/image10.png
  :align: center
  :scale: 70%

Restoring the System Controller from a Database Backup
======================================================

Restoring the System Controller from a Database Backup via CLI
--------------------------------------------------------------


Now that the system controller backup has been copied into the system, you can restore the previous backup using the **system database config-restore** command as seen below. You can use the **file list** command to verify the file name:

.. code-block:: bash

    syscon-2-active# file list path configs/ 
    entries {
        name 
    SYSTEM-CONTROLLER-DB-BACKUP2021-09-10
    }
    syscon-2-active# 


    syscon-2-active(config)# system database config-restore name SYSTEM-CONTROLLER-DB-BACKUP2021-09-10
    response Succeeded.
    syscon-2-active(config)#

Restoring the System Controller from a Database Backup via API
--------------------------------------------------------------

To restore the system controller ConfD database use the following API call:

.. code-block:: bash

    POST https://{{velos_chassis1_system_controller_ip}}:8888/restconf/data/openconfig-system:system/f5-database:database/f5-database:config-restore

.. code-block:: json

    {
    "f5-database:name": "SYSTEM-CONTROLLER-DB-BACKUP{{currentdate}}"
    }

Restoring the System Controller from a Database Backup via webUI
--------------------------------------------------------------

Currently there is no webUI support for restoration of the ConfD database, so you’ll need to use either the CLI or API to restore the system controller’s database. Once the database has been restored (you may need to wait a few minutes for the restoration to complete.) you need to reboot the blades in-order for the config to be deployed successfully.

To reboot blades from the webUI log into each chassis partition. You will be prompted to change the password on first login. 

.. image:: images/velos_f5os_configuration_backup_and_restore/image11.png
  :align: center
  :scale: 70%

Once logged in you’ll notice no configuration inside the chassis partition. Go to the **System Settings -> General** Page and reboot each blade. You’ll need to do the same procedure for other chassis partitions if they exist.

.. image:: images/velos_f5os_configuration_backup_and_restore/image12.png
  :align: center
  :scale: 70%


Wait for each blade to return to the **Ready** status before going onto the next step.

To reboot blades from the API, using the following API commands to list nodes (Blades), and then reboot them. The command below will list the current nodes and their names that can then be used to reboot. Send the API call to the chassis partition IP address:

.. code-block:: bash

    GET https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/f5-cluster:cluster/nodes

.. code-block:: json

    {
        "f5-cluster:nodes": {
            "node": [
                {
                    "name": "blade-1",
                    "config": {
                        "name": "blade-1",
                        "enabled": true
                    },
                    "state": {
                        "name": "blade-1",
                        "enabled": true,
                        "node-running-state": "running",
                        "assigned": true,
                        "platform": {
                            "fpga-state": "FPGA_RDY",
                            "dma-agent-state": "DMA_AGENT_RDY"
                        },
                        "slot-number": 1,
                        "node-info": {
                            "creation-time": "2021-08-31T00:16:13Z",
                            "cpu": 28,
                            "pods": 250,
                            "memory": "131574100Ki"
                        },
                        "ready-info": {
                            "ready": true,
                            "last-transition-time": "2021-09-16T00:36:42Z",
                            "message": "kubelet is posting ready status"
                        },
                        "out-of-disk-info": {
                            "out-of-disk": false,
                            "last-transition-time": "2021-09-16T00:36:31Z",
                            "message": "kubelet has sufficient disk space available"
                        },
                        "disk-pressure-info": {
                            "disk-pressure": false,
                            "last-transition-time": "2021-09-16T00:36:31Z",
                            "message": "kubelet has no disk pressure"
                        },
                        "disk-data": {
                            "stats": [
                                {},
                                {},
                                {}
                            ]
                        },
                        "f5-disk-usage-threshold:disk-usage": {
                            "used-percent": 1,
                            "growth-rate": 0,
                            "status": "in-range"
                        }
                    }
                },
                {
                    "name": "blade-2",
                    "config": {
                        "name": "blade-2",
                        "enabled": true
                    },
                    "state": {
                        "name": "blade-2",
                        "enabled": true,
                        "node-running-state": "running",
                        "assigned": true,
                        "platform": {
                            "fpga-state": "FPGA_RDY",
                            "dma-agent-state": "DMA_AGENT_RDY"
                        },
                        "slot-number": 2,
                        "node-info": {
                            "creation-time": "2021-08-31T00:16:12Z",
                            "cpu": 28,
                            "pods": 250,
                            "memory": "131574100Ki"
                        },
                        "ready-info": {
                            "ready": true,
                            "last-transition-time": "2021-09-16T00:36:44Z",
                            "message": "kubelet is posting ready status"
                        },
                        "out-of-disk-info": {
                            "out-of-disk": false,
                            "last-transition-time": "2021-09-16T00:36:34Z",
                            "message": "kubelet has sufficient disk space available"
                        },
                        "disk-pressure-info": {
                            "disk-pressure": false,
                            "last-transition-time": "2021-09-16T00:36:34Z",
                            "message": "kubelet has no disk pressure"
                        },
                        "disk-data": {
                            "stats": [
                                {},
                                {},
                                {}
                            ]
                        },
                        "f5-disk-usage-threshold:disk-usage": {
                            "used-percent": 1,
                            "growth-rate": 0,
                            "status": "in-range"
                        }
                    }
                },
                {
                    "name": "blade-3",
                    "config": {
                        "name": "blade-3",
                        "enabled": true
                    },
                    "state": {
                        "name": "blade-3",
                        "enabled": true,
                        "node-running-state": "running",
                        "assigned": false,
                        "slot-number": 3
                    }
                },
                {
                    "name": "blade-4",
                    "config": {
                        "name": "blade-4",
                        "enabled": true
                    },
                    "state": {
                        "name": "blade-4",
                        "enabled": true,
                        "node-running-state": "running",
                        "assigned": false,
                        "slot-number": 4
                    }
                },
                {
                    "name": "blade-5",
                    "config": {
                        "name": "blade-5",
                        "enabled": true
                    },
                    "state": {
                        "name": "blade-5",
                        "enabled": true,
                        "node-running-state": "running",
                        "assigned": false,
                        "slot-number": 5
                    }
                },
                {
                    "name": "blade-6",
                    "config": {
                        "name": "blade-6",
                        "enabled": true
                    },
                    "state": {
                        "name": "blade-6",
                        "enabled": true,
                        "node-running-state": "running",
                        "assigned": false,
                        "slot-number": 6
                    }
                },
                {
                    "name": "blade-7",
                    "config": {
                        "name": "blade-7",
                        "enabled": true
                    },
                    "state": {
                        "name": "blade-7",
                        "enabled": true,
                        "node-running-state": "running",
                        "assigned": false,
                        "slot-number": 7
                    }
                },
                {
                    "name": "blade-8",
                    "config": {
                        "name": "blade-8",
                        "enabled": true
                    },
                    "state": {
                        "name": "blade-8",
                        "enabled": true,
                        "node-running-state": "running",
                        "assigned": false,
                        "slot-number": 8
                    }
                }
            ]
        }
    }

You must reboot each blade that was previously assigned to a partition:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/f5-cluster:cluster/nodes/node=blade-1/reboot

    POST https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/f5-cluster:cluster/nodes/node=blade-2/reboot

    POST https://{{velos_chassis1_chassis_partition2_ip}}:8888/restconf/data/f5-cluster:cluster/nodes/node=blade-3/reboot




Importing Archived Chassis Partition Configs
============================================


Importing Archived Chassis Partition Configs via CLI
----------------------------------------------------


Log directly into the chassis partition CLI and use the **file import** command to copy the archived image from a remote HTTPS server. You can then use the **file transfer-status** to see if the import succeeded, and then the **file list** command to see the file.

.. code-block:: bash

    Production-1# file import remote-host 10.255.0.142 remote-file /upload/Production-DB-BACKUP2021-09-10 local-file configs/Production-DB-BACKUP2021-09-10 username corpuser insecure  
    Value for 'password' (<string>): ********
    result File transfer is initiated.(configs/Production-DB-BACKUP2021-09-10)


    Production-1# file transfer-status 
    result 
    S.No.|Operation  |Protocol|Local File Path                                             |Remote Host         |Remote File Path                                            |Status            |Time                
    1    |Import file|HTTPS   |configs/Production-DB-BACKUP2021-09-10                    |10.255.0.142        |/upload/Production-DB-BACKUP2021-09-10                    |         Completed|Wed Sep 15 03:15:43 2021



    Production-1# file list path configs/
    entries {
        name 
    Production-DB-BACKUP2021-09-10
    }
    Production-1# 

Repeat this process for each chassis partition in the system.

.. code-block:: bash

    development-1# file import remote-host 10.255.0.142 remote-file /upload/development-DB-BACKUP2021-09-10 local-file configs/development-DB-BACKUP2021-09-10 username corpuser insecure 
    Value for 'password' (<string>): ********
    result File transfer is initiated.(configs/development-DB-BACKUP2021-09-10)


    development-1# file transfer-status 
    result 
    S.No.|Operation  |Protocol|Local File Path                                             |Remote Host         |Remote File Path                                            |Status            |Time                
    1    |Import file|HTTPS   |configs/development-DB-BACKUP2021-09-10                  |10.255.0.142        |/upload/development-DB-BACKUP2021-09-10                  |         Completed|Wed Sep 15 03:21:40 2021



    development-1# file list path configs/
    entries {
        name 
    development-DB-BACKUP2021-09-10
    }
    development-1# 

Importing Archived Chassis Partition Configs via API
----------------------------------------------------

Archived ConfD database backups can be imported from a remote HTTPS, SFTP, or SCP server via the following API call to the chassis partition IP addresses. Each chassis partition will need to have its own archived database imported so that it may be restored:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition2_ip}}:8888/restconf/data/f5-utils-file-transfer:file/import

.. code-block:: json

    {
        "f5-utils-file-transfer:insecure": "",
        "f5-utils-file-transfer:protocol": "https",
        "f5-utils-file-transfer:username": "corpuser",
        "f5-utils-file-transfer:password": "Passw0rd!!",
        "f5-utils-file-transfer:remote-host": "10.255.0.142",
        "f5-utils-file-transfer:remote-file": "/upload/development-DB-BACKUP2021-09-10",
        "f5-utils-file-transfer:local-file": "configs/development-DB-BACKUP2021-09-10"
    }

You can check on the file transfer status by issubg the following API call:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition1_ip}}:8888/api/data/f5-utils-file-transfer:file/transfer-status

A status similar to the one below will show a status of completed if successful:

.. code-block:: json

    {
        "f5-utils-file-transfer:output": {
            "result": "\nS.No.|Operation  |Protocol|Local File Path                                             |Remote Host         |Remote File Path                                            |Status            |Time                \n1    |Import file|HTTPS   |configs/Production-DB-BACKUP2021-09-10                    |10.255.0.142        |/upload/Production-DB-BACKUP2021-09-10                    |         Completed|Thu Sep 16 01:33:50 2021"
        }
    }

Repeat similar steps for remaining chassis partitions:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/f5-utils-file-transfer:file/import

.. code-block:: json

    {
        "f5-utils-file-transfer:insecure": "",
        "f5-utils-file-transfer:protocol": "https",
        "f5-utils-file-transfer:username": "corpuser",
        "f5-utils-file-transfer:password": "Passw0rd!!",
        "f5-utils-file-transfer:remote-host": "10.255.0.142",
        "f5-utils-file-transfer:remote-file": "/upload/Production-DB-BACKUP2021-09-10",
        "f5-utils-file-transfer:local-file": "configs/Production-DB-BACKUP2021-09-10"
    }

Importing Archived Chassis Partition Configs via webUI
----------------------------------------------------

You can use the **System Settings -> File Utilities** page to import archives from a remote HTTPS server. 

.. image:: images/velos_f5os_configuration_backup_and_restore/image13.png
  :align: center
  :scale: 70%

Restoring Chassis Partitions from Database Backups
==================================================

To restore a configuration database backup within a chassis partition, use the **system database config-restore** command inside the chassis partition. Note that a newly restored chassis partition will not have any tenant images loaded so tenants will show a **Pending** status until the proper image is loaded for that tenant.

.. code-block:: bash

    Production-1(config)# system database config-restore name Production-DB-BACKUP2021-09-10
    A clean configuration is required before restoring to a previous configuration.
    Please perform a reset-to-default operation if you have not done so already.
    Proceed? [yes/no]: yes
    result Database config-restore successful.
    Production-1(config)# 
    System message at 2021-09-15 03:25:53...
    Commit performed by admin via tcp using cli.
    Production-1(config)# 


    Development-1(config)# system database config-restore name development-DB-BACKUP2021-09-10
    A clean configuration is required before restoring to a previous configuration.
    Please perform a reset-to-default operation if you have not done so already.
    Proceed? [yes/no]: yes
    result Database config-restore successful.
    Development-1(config)# 
    System message at 2021-09-15 03:23:50...
    Commit performed by admin via tcp using cli.
    Development-1(config)# 


The tenant is properly restored and deployed; however, its status is pending waiting on image:


.. image:: images/velos_f5os_configuration_backup_and_restore/image14.png
  :align: center
  :scale: 70%

This can be seen in the chassis partition CLI by using the **show tenants** command. Note the **Phase** will display: **Tenant image not found**.

.. code-block:: bash

    Placeholder

Copy the proper tenant image into each partition and the tenant should then deploy successfully. Below is a **show images** output before and after an image is successfully uploaded. Note the **STATUS** of **not-present** and then **replicated** after the image has been uploaded:   

 .. code-block:: bash

    Production-1# show images 
                                                    IN                  
    NAME                                            USE    STATUS       
    --------------------------------------------------------------------
    BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle  false  not-present  


    Production-1# show images
                                                    IN                 
    NAME                                            USE    STATUS      
    -------------------------------------------------------------------
    BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle  false  replicated  

Once the tenant is deployed you may login, and the upload and restore the tenant UCS image.

Restoring Chassis Partitions from Database Backups via API
----------------------------------------------------------

The following API commands will restore the database backups on the two chassis partitions:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/openconfig-system:system/f5-database:database/f5-database:config-restore

.. code-block:: json

    {
    "f5-database:name": "Production-DB-BACKUP2021-09-10"
    }

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition2_ip}}:8888/restconf/data/openconfig-system:system/f5-database:database/f5-database:config-restore

.. code-block:: json

    {
    "f5-database:name": "development-DB-BACKUP2021-09-10"
    }

The tenants are properly restored and deployed; however, its status is pending waiting on image. You can check the status of the images with the following API call:

.. code-block:: bash

    GET https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/f5-tenant-images:images

You will need to load the image that the tenant was running when it was archived. The following API call will import a tenant image from a remote HTTPS server:

.. code-block:: bash

    POST https://{{velos_chassis1_chassis_partition1_ip}}:8888/api/data/f5-utils-file-transfer:file/import

.. code-block:: json

    {
        "input": [
            {
                "remote-host": "10.255.0.142",
                "remote-file": "upload/{{Tenant_Image}}",
                "local-file": "images/{{Tenant_Image}}",
                "insecure": "",
                "f5-utils-file-transfer:username": "corpuser",
                "f5-utils-file-transfer:password": "Passw0rd!!"
            }
        ]
    }

You can verify the tenant has successfully started once the image has been loaded:

.. code-block:: bash

    GET https://{{velos_chassis1_chassis_partition1_ip}}:8888/restconf/data/f5-tenants:tenants

.. code-block:: json

    {
        "f5-tenants:tenants": {
            "tenant": [
                {
                    "name": "tenant1",
                    "config": {
                        "name": "tenant1",
                        "type": "BIG-IP",
                        "image": "BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle",
                        "nodes": [
                            1
                        ],
                        "mgmt-ip": "10.255.0.149",
                        "prefix-length": 24,
                        "gateway": "10.255.0.1",
                        "vlans": [
                            501,
                            3010,
                            3011
                        ],
                        "cryptos": "enabled",
                        "vcpu-cores-per-node": "4",
                        "memory": "14848",
                        "storage": {
                            "size": 76
                        },
                        "running-state": "deployed",
                        "appliance-mode": {
                            "enabled": false
                        }
                    },
                    "state": {
                        "name": "tenant1",
                        "unit-key-hash": "Y00du3mZxvi0UXGNV32NpCMLTRia8AbLvaHwAAuLxg2IS6EWppPwnSGSecfleaHh0lHXENQWKACz27xe9CyW5w==",
                        "type": "BIG-IP",
                        "image": "BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle",
                        "nodes": [
                            1
                        ],
                        "mgmt-ip": "10.255.0.149",
                        "prefix-length": 24,
                        "gateway": "10.255.0.1",
                        "mac-ndi-set": [
                            {
                                "ndi": "default",
                                "mac": "00:94:a1:8e:d0:0b"
                            }
                        ],
                        "vlans": [
                            501,
                            3010,
                            3011
                        ],
                        "cryptos": "enabled",
                        "vcpu-cores-per-node": "4",
                        "memory": "14848",
                        "storage": {
                            "size": 76
                        },
                        "running-state": "deployed",
                        "mac-data": {
                            "base-mac": "00:94:a1:8e:d0:09",
                            "mac-pool-size": 1
                        },
                        "appliance-mode": {
                            "enabled": false
                        },
                        "status": "Running",
                        "primary-slot": 1,
                        "image-version": "BIG-IP 15.1.4 0.0.46",
                        "instances": {
                            "instance": [
                                {
                                    "node": 1,
                                    "instance-id": 1,
                                    "phase": "Running",
                                    "image-name": "BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle",
                                    "creation-time": "2021-09-16T01:57:11Z",
                                    "ready-time": "2021-09-16T01:56:58Z",
                                    "status": "Started tenant instance",
                                    "mgmt-mac": "36:4d:6d:2d:a8:80"
                                }
                            ]
                        }
                    }
                },
                {
                    "name": "tenant2",
                    "config": {
                        "name": "tenant2",
                        "type": "BIG-IP",
                        "image": "BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle",
                        "nodes": [
                            1,
                            2
                        ],
                        "mgmt-ip": "10.255.0.205",
                        "prefix-length": 24,
                        "gateway": "10.255.0.1",
                        "vlans": [
                            502,
                            3010,
                            3011
                        ],
                        "cryptos": "enabled",
                        "vcpu-cores-per-node": "6",
                        "memory": "22016",
                        "storage": {
                            "size": 76
                        },
                        "running-state": "deployed",
                        "appliance-mode": {
                            "enabled": false
                        }
                    },
                    "state": {
                        "name": "tenant2",
                        "unit-key-hash": "fRO3SmBcQxURAjrANfv8u4J9EDH+kG1KevOn99rvDupNW2HMyoBeWqN4nhabnmAha/wbbNxAR9l2JW9LEF+7FQ==",
                        "type": "BIG-IP",
                        "image": "BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle",
                        "nodes": [
                            1,
                            2
                        ],
                        "mgmt-ip": "10.255.0.205",
                        "prefix-length": 24,
                        "gateway": "10.255.0.1",
                        "mac-ndi-set": [
                            {
                                "ndi": "default",
                                "mac": "00:94:a1:8e:d0:0c"
                            }
                        ],
                        "vlans": [
                            502,
                            3010,
                            3011
                        ],
                        "cryptos": "enabled",
                        "vcpu-cores-per-node": "6",
                        "memory": "22016",
                        "storage": {
                            "size": 76
                        },
                        "running-state": "deployed",
                        "mac-data": {
                            "base-mac": "00:94:a1:8e:d0:0a",
                            "mac-pool-size": 1
                        },
                        "appliance-mode": {
                            "enabled": false
                        },
                        "status": "Running",
                        "primary-slot": 1,
                        "image-version": "BIG-IP 15.1.4 0.0.46",
                        "instances": {
                            "instance": [
                                {
                                    "node": 1,
                                    "instance-id": 1,
                                    "phase": "Running",
                                    "image-name": "BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle",
                                    "creation-time": "2021-09-16T01:58:41Z",
                                    "ready-time": "2021-09-16T01:58:27Z",
                                    "status": "Started tenant instance",
                                    "mgmt-mac": "de:08:94:a8:1b:08"
                                },
                                {
                                    "node": 2,
                                    "instance-id": 2,
                                    "phase": "Running",
                                    "image-name": "BIGIP-15.1.4-0.0.46.ALL-VELOS.qcow2.zip.bundle",
                                    "creation-time": "2021-09-16T01:58:37Z",
                                    "ready-time": "2021-09-16T01:58:24Z",
                                    "status": "Started tenant instance",
                                    "mgmt-mac": "a6:fe:75:70:21:c8"
                                }
                            ]
                        }
                    }
                }
            ]
        }
    }


The final step is to restore the backups on each individual tenant. This will follow the normal BIG-IP UCS restore process.
