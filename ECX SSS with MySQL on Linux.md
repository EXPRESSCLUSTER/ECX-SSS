# EXPRESSCLUSTER X SingleServerSafe with MySQL Database on Linux.

This guide describes how to setup EXPRESSCLUSTER X SingleServerSafe with MySQL Database on RedHat Linux Environment. 
EXPRESSCLUSTER X SingleServerSafe is set up on a server. It monitors for application errors and hardware failures on the server and, upon detecting an error or failure, restarts the failed application or reboots the server to ensure greater server availability.
May refer to [this site](https://www.nec.com/expresscluster) for details of ECX itself.

---


### Software versions

- RedHat 8.6
- MySQL 8.0.32
- EXPRESSCLSSSS 5.1.1

### System configuration

```bat
<Public LAN>
 |
 | <Private LAN>
 |  |
 |  |  +----------------------------------------------+
 +-----| Server (10.0.7.121)                          |
 |  |  |  Red Hat Enterprise Linux release 8.6 (Ootpa)|           
 |  |  |  EXPRESSCLSSSS 5.1                           |
 |  +--|  MySQL 8.0.32                                |
 |  |  +----------------------------------------------+
 |  |
 |
[Gateway]
 :
```

### Cluster installation

1. Unzip the downloaded EXPRESSCLUSTER X SingleServerSafe file 

    ```
    [root@Linuxms ~]# unzip ecxsss51l_x64.zip
    ```
2. Browse to your EXPRESSCLUSTER X SingleServerSafe Package file

    ```
    [root@Linuxms ~]# cd ecxsss51l_x64/Linux/5.1/en/server/
    [root@Linuxms server]# ls
    expressclssss-5.1.1-1.x86_64.rpm
    ```

3. Install EXPRESSCLUSTER X SingleServerSafe

    ```
    [root@Linuxms server]# rpm -ivh expressclssss-5.1.1-1.x86_64.rpm
    ```

4. Configure EXPRESSCLUSTER X SingleServerSafe required Licenses

### Cluster configuration

- #### **Group resources**
  - **exec-webser (exec resource)**: Manages Http server.
  - **exec-mysql (exec resource)**: Manages MySQL services.
- #### **Monitor resources**
  - **diskw-SSS (disk monitor resource)**: Monitors the SSS Server device by using the WRITE(FILE) monitoring.
  - **genw-mysql (custom monitor resource)** : Monitors the status of MySQL service.
  - **ipw (ip monitor resource)**: Monitors the status of IP addresses by using the ping command.
  - **miiw (nic link up/down monitor resource)**: Monitors the status of NIC Link Up/Down.
  - **mtw (multi target monitor resource)**: Monitors the status of more than one monitor resources.
  - **mysqlw (MySQL monitor resource)**: Monitors the status pf MySQL database that operates on servers.
  - **psrw (Process resource monitor resource)**: Monitors the amounts of process resources used and then analyze the amounts.
  - **psw (Process name monitor resource)**: Monitors the process of specified processes.
  - **sraw (System monitor resource)**: Periodically collect the amounts of system resources and disk resources used and then analyze the amounts.
  - **volmgrw (Volume manager monitor resources)**: Monitors the status of the logical disks managed by the volume manager.

### Group Resources
1. #### Configure the failover group "failover-SSS".

1. #### Configure the exec resource "exec-Webser".
   - Details
     - Start script:
       ```sh
       #! /bin/sh
       #**************************************
       #*              start.sh               *
       #***************************************

       python3 -m http.server 8080
       ```
     - Stop script:
       ```sh
       #! /bin/sh
       #***************************************
       #*               stop.sh               *
       #***************************************

       killall python3
       ```
1. #### Configure the exec resource "exec-mysql".
   - Details
     - Start script:
       ```sh
       #! /bin/sh
       #***************************************
       #*              start.sh               *
       #***************************************
       
       Sudo systemctl start mysql.service
       ```
     - Stop script:
       ```sh
       #! /bin/sh
       #***************************************
       #*               stop.sh               *
       #***************************************
       
       Sudo systemctl stop mysql.service
       ```

### Monitor Resources

1. #### Configure the disk monitor resource "diskw-SSS".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: exec-mysql)
   - Monitor(special)
     - Method: READ(0_DIRECT)
     - Monitor Target: /dev/sda1
   - Recovery Action
     - Recovery Action: Restart the recovery target, and if there is no effect with restart, then failover
     - Recovery Target: failover-SSS

1. #### Configure the custom monitor resource "genw-mysql".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: exec-mysql)
   - Monitor(special)
     - Monitor script:
      
       ```sh
       #! /bin/sh
       #***********************************************
       #*                   genw.sh                   *
       #*********************************************** 
       name=mysqld.service
       systemctl status $name
       ret=$?
       if [ $ret = 0 ]; then
       echo $name is running.
       exit 0
       else
       echo $name is NOT running.
       clplogcmd -m "$name is NOT running."
       exit 1
       fi
       ```
   - Recovery Action
     - Recovery Action: Custom setting
     - Recovery Target: LocalServer
     - Recovery Script Execution Count: 1
     - Recovery script:
       ```sh
       #! /bin/sh
       #***********************************************
       #*	          preaction.sh                 *
       #***********************************************

       systemctl restart mysqld.service
       result=$?z
       echo "exit status: $result"
       exit $result
       ```
     - Final Action: Stop the cluster service and reboot OS


1. #### Configure the IP monitor resource "ipw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Add IP address List:
     ```
     10.0.7.174
     10.0.7.178
     10.0.7.175
     ```
   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: No Operation

1. #### Configure the NIC Link Up/Down monitor resource "miiw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Monitor Target: ens192
   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: Stop the cluster service and reboot OS

1. #### Configure the Multi target monitor resource "mtw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Add Monitor resources from Available Monitor Resources:
     ```
     Monitor Resources                      
     Monitor Resource	Type                        Available Monitor Resources
     diskw	            diskw       <--ADD          Monitor Resource	Type
     genw	            genw        REMOVE-->       psrw	            psrw
     miiw	            miiw                        sraw	            sraw
     ```
     - Tuning
        - Failure Threshold: Specify Number= 3
        - Warning Threshold: Specify Number= 1

   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: Stop the cluster service and reboot OS


1. #### Configure the MySQL monitor resource "mysqlw".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: exec-mysql)

   - Monitor(special)
     - Monitor Level: Level 3 (create/drop table each time)
     - Database Name: testdb
     - IP Address: 10.0.7.121
     - Port: 3306
     - User Name: test_user
     - Password: "password123"
     - Table: mysqlwatch
     - Storage Engine: InnoDB
     - Library Path: /usr/lib64/mysql/libmysqlclient.so.21
     
   - Recovery Action
     - Recovery Action: Custom setting
     - Recovery Target: LocalServer
     - Recovery Script Execution Count: 1
     - Recovery script:
       ```sh
       #! /bin/sh
       #***********************************************
       #*	          preaction.sh                 *
       #***********************************************

       systemctl restart mysqld.service
       result=$?
       echo "exit status: $result"
       exit $result
       ```
     - Final Action: Stop the cluster service and reboot OS

1. #### Configure the Process resource monitor resource "psrw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Specify the process monitoring conditions for identifying failure 
        - Process Name: systemd
        - Monitoring CPU usage: check
            - CPU usage: 90
            - Duration Time: 1440
        - Monitoring usage of memory: check
            - Rate of Increase from the First Monitoring Point: 10
            - Maximum Refresh Count: 1440 
        - Monitoring number of opening files(maximum number): check
            - Refresh Count: 1000
        - Monitoring number of opening files(kernel limit)
            - Ratio: 90
        - Monitoring number of running threads: check
            - Duration Time: 1440
        - Monitoring Zombie Processes: check
            - Duration Time: 1440 
        - Monitoring Processes of the Same Name
            - Count: 05
   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: Stop the cluster service and reboot OS

1. #### Configure the Process name monitor resource "psw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Process Name: /usr/libexec/mysqld
     - Minimum Process Count: 1
   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: mysqlser
     - Final Action: Stop the cluster service and reboot OS

1. #### Configure the System monitor resource "sraw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Specify the system monitoring conditions for identifying abnormality
        - Monitoring CPU usage: check
            - CPU usage: 90
            - Duration Time: 60
        - Monitoring total usage of memory: check
            - Total usage of memory: 90
            - Duration Time: 60
        - Monitoring total usage of virtual memory: check
            - Total usage of virtual memory: 90
            - Duration Time: 60
        - Monitoring total number of opening files: check
            - Total number of opening files (in a ratio comparing with the system upper limit): 90
            - Duration Time: 60
        - Monitoring number of running threads: check
            - Total number of running threads: 90
            - Duration Time: 60
        - Monitoring number of running process of each user: check
            - Number of running process of each user: 90
            - Duration Time: 60
        ```
        Condition of detecting failure
        Warning:When exceeding level once
        Notification:When continuously exceeding level over the duration
        ```

        - Monitoring target disk list
          - Add Monitoring target disk list
            - Mount Point: /mnt/test
            - Monitor Type: Default

   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: Local Server
     - Final Action: Stop the cluster service and reboot OS

1. #### Configure the Volume manager monitor resources "volmgrw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Volume Manager: lvm
     - Target Name: rhel
   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: Stop the cluster service and reboot OS

1. Apply the cluster configuration.
