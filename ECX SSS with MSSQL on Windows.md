# EXPRESSCLUSTER X SingleServerSafe with MSSQL Database on Windows.

This guide describes how to setup EXPRESSCLUSTER X SingleServerSafe with MSSQL Database on Windows Server 2022 Environment. 
EXPRESSCLUSTER X SingleServerSafe is set up on a server. It monitors for application errors and hardware failures on the server and, upon detecting an error or failure, restarts the failed application or reboots the server to ensure greater server availability.
May refer to [this site](https://www.nec.com/expresscluster) for details of ECX itself.

---

### Software versions

- Windows Server 2022 Standard
- MSSQL 2022
- EXPRESSCLUSTER X SingleServerSafe 5.1.1

### System configuration

```bat
<Public LAN>
 |
 | <Private LAN>
 |  |
 |  |  +----------------------------------------------+
 +-----| Server (10.0.7.195)                          |
 |  |  |  Windows Server 2022 Standard                |           
 |  |  |  EXPRESSCLUSTER X SingleServerSafe 5.1                           |
 |  +--|  MSSQL 2022                                  |
 |  |  +----------------------------------------------+
 |  |
 |
[Gateway]
 :
```
### EXPRESSCLUSTER X SingleServerSafe installation procedure

Please refer https://www.nec.com/en/global/prod/expresscluster/en/doc/manuals/W51_SSS_IG_EN_04.pdf


### Cluster configuration
#### Group Resource 
  - **MSSQLSRV (Service Resource)**: Manages SQLSERVER service 
  - **SQLAGENTS (Service Resource)**: Manages SQLAgent Service 

#### Monitor resources
  - **servicew1 (MSSQL Monitor)**: Monitors the status of MSSQL service.
  - **servicew2 (MSSQLAGENT Monitor)**: Monitors the status of MSSQL Agent service.
  - **diskw (Disk RW monitor)**: Monitors the SSS Server device by using the WRITE(FILE) monitoring.
  - **ipw (iP Monitor)**: Monitors the status of IP addresses by using the ping command.
  - **miiw(Nic LInk UpDown Monitor)**: Monitors the status of NIC Link Up/Down.
  - **mtw (multi Target Monitor)**: Monitors the status of more than one monitor resources.
  - **sqlserverw (SQL Server Monitor)**: Monitors the status pf MSSQL database that operates on server.
  - **psrw (Process Resource Monitor)**: Monitors the amounts of process resources used and then analyze the  amounts.
  - **psw (Process Name Monitor)**: Monitors the process of specified processes.
  - **sraw (System Monitor)**: Periodically collect the amounts of system resources and disk resources used and then analyze the amounts.

###  Group resource configuration
#### Open cluster WebUI config mode and configure the failover group "SQLSRVRGRP".

1. #### Configure MSSQL service resource "mssqlsrv".
	  - Click on Add resource botton (+) under failover group "SQLSRVRGRP".
		 - Info
			- Type: service resource
			- Name: service_SQLServer
		- Dependency  
			Default
		- Recovery Operation  
			Default or as you like
		- Details  
			Click connect and select [SQL Server (<instance name>)]

1. #### Configure MSSQL Agent service resource "service_SQLAgent".
    - Click on Add resource botton (+) under failover group "SQLSRVRGRP".
			- Type: service resource
			- Name: service_SQLAgent
		- Dependency  
			Uncheck [Follow the default dependency], select [service_SQLServer] and click [<Add]
		- Recovery Operation  
			Default or as you like
		- Details  
			Click connect and select [SQL Server Agent (<instance name>)]
1. Apply the configuration
1. Start the resources on EXPRESSCLUSTER X SingleServerSafe Server.

### Add monitor resource
#### Start cluster WebUI config mode

1. #### Configure SQL Server service monitor resource "Servicew1" to existing failover group "SQLSRVRGRP".
	 - Click on Add Monitor Resource botton (+) under failover group "SQLSRVRGRP".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: MSSQLSRV)
   - Monitor(special)
     - Add Service Name "SQL Server(MSSQLSERVER)
   - Recovery Action
     - Recovery Action: Custom Settings
     - Recovery Target: MSSQLSRV
     - Final Action: No Operation

1. #### Configure the SQLAgent service monitor resource "Servicew2".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: sqlagentservice)
   - Monitor(special)
     - Add Service Name "SQL Serverv Agent(MSSQLSERVER)
   - Recovery Action
     - Recovery Action: Custom Settings
     - Recovery Target: sqlagentservice
     - Final Action: No Operation

1. #### Configure the disk monitor resource "diskw-SSS".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: MSSQL)
   - Monitor(special)
     - File Name: d:\chk.txt
     - I/O Size: 2000000 byte
     - Action on Stall: No Operation or as you like
     - Action When Diskfull Is Detected: Recover / Do not Recover
     - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: No Operation


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
     - Monitor Target: 10.0.7.195 (Testing server IP)
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
     diskw	           diskw       <--ADD        Monitor Resource	    Type
     ipw	               ipw        REMOVE-->       servicew1           servicew
     miiw	           miiw                       servicew2           servicew
     psrw	           psrw                       sqlserverw          sqlserverw
     sraw	           sraw
     psw                psw
     ```
     - Tuning
        - Failure Threshold: Specify Number= 3
        - Warning Threshold: Specify Number= 1

   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: Stop the cluster service and reboot OS

1. #### Configure the Process resource monitor resources "psrw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Specify the process monitoring conditions for identifying failure 
        - Process Name: msedge.exe
        - Monitoring CPU usage: check
            - CPU usage: 90
            - Duration Time: 1440
        - Monitoring usage of memory: check
            - Rate of Increase from the First Monitoring Point: 10
            - Maximum Refresh Count: 10 
        - Monitoring number of opening files(maximum number): check
            - Refresh Count: 1440
        - Monitoring number of running threads: check
            - Duration Time: 1440
        - Monitoring Zombie Processes: check
            - Duration Time: 1440 
        - Monitoring Processes of the Same Name
            - Count: 100
   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: Stop the cluster service and reboot OS

1. #### Configure the MSSQL monitor resource "sqlserverw".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: MSSQL)

   - Monitor(special)
     - Monitor Level: Level 2 (monitoring by update/select)
     - Database Name: test
     - Instance: MSSQLSERVER
     - User Name: SA
     - Password: "password123"
     - Monitor Table Name: sqlwatch
     - ODBC Driver Name: ODBC Driver 17 for SQL Server
     
   - Recovery Action
     - Recovery Action: Custom setting
     - Recovery Target: sqlsrvrgrp
     - Final Action: Stop the cluster service and reboot OS


1. #### Configure the Process name monitor resource "psw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Process Name: "C:\Program Files\Microsoft SQL Server\90\Shared\sqlwriter.exe"
     - Minimum Process Count: 1
   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: sqlsrvrgrp
     - Final Action: Stop the cluster service and reboot OS

1. #### Configure the System monitor resource "sraw".
   - Monitor(common)
     - Monitor Timing: Always
   - Monitor(special)
     - Specify the system monitoring conditions for identifying abnormality
        - Monitoring CPU usage: check
            - CPU usage: 80
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

        ```
        Condition of detecting failure
        Warning:When exceeding level once
        Notification:When continuously exceeding level over the duration
        ```

        - Monitoring target disk list
          - Add Monitoring target disk list
            - Logical Drive: C:\ or you can set
            - Warning (%): Default or you can set
            - Notification (%): 80 or you can set
            - Duration Time(min):1440 or you can set
            - Warning(MB): 500 or you can set
            - Notification(MB): 1000 or you can set
            - Duration Time(min): 1440 or you can set

       - Recovery Action
            - Recovery Action: Execute only the final action
            - Recovery Target: Local Server
            - Final Action: Stop the cluster service and reboot OS

     
1. #### Apply the cluster configuration.

