--------

# SYBASE HADR on SLE HAE (pacemaker) manual deployment and configuration  

--------


- Author: Chandru N N
- Email: chandrun@in.ibm.com

- Date: 2018/05/30
- Version: 01

# Table of contents

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [SYBASE HADR on SLE HAE (pacemaker) manual deployment and configuration](#sybase-hadr-on-sle-hae-pacemaker-manual-deployment-and-configuration)
- [Table of contents](#table-of-contents)
- [Introduction](#introduction)
- [Values / parameters used in this document](#values-parameters-used-in-this-document)
- [Used Cluster Nodes](#used-cluster-nodes)
- [Prerequisites](#prerequisites)
	- [Cluster](#cluster)
	- [Middleware](#middleware)
- [Configure cluster Properties and Resources](#configure-cluster-properties-and-resources)
	- [Set cluster in maintenance mode](#set-cluster-in-maintenance-mode)
	- [Define required resources](#define-required-resources)
- [the IP resource](#the-ip-resource)
- [the DB resource, note the additional monitor op with role "Master"](#the-db-resource-note-the-additional-monitor-op-with-role-master)
- [the m/s resource, notifications are required](#the-ms-resource-notifications-are-required)
- [location information](#location-information)
	- [Activate the cluster](#activate-the-cluster)
- [Add the Replication Details in Sybase env](#Add-the-Replication-Details-in-Sybase-env)
- [Useful commands and information](#useful-commands-and-information)
	- [Clear SBD device after Node crash](#clear-sbd-device-after-node-crash)
- [Reference and links](#reference-and-links)

<!-- /TOC -->



# Introduction

This document is intended for general guidance how to deploy and configure SLE HAE Cluster on CMS 3.x (_Softlayer environment_).
It considers a two cluster solution using one cluster for _Sybase HADR_ and the second cluster for _SAP Application Server_.
This document handles the deployment and configuration of _Sybase HADR_.

Deployment and configuration is demonstrated on an example.
Commands and configuration files must be adjusted for specific case before using them.


&#x2714; **NOTE**  

- _SAP Application Server_ and _SAP AS_ will be used interchangeably
- _server_ and _node_ will be used interchangeably

# Values / parameters used in this document

|Parameter		|Value 						|Note|
|:---------:			|:-----:			|:-----|
| OS Linux		|SLES 12 SP2 for SAP|SUSE Linux Enterprise Server for SAP Applications 12 SP2|
| pacemaker version		|1.1.15-21.1 |pacemaker-1.1.15-21.1.x86_64|
| SAP SID				|SA1				|  	|
| SBD device   | /dev/mapper/36000c29b9d40a7576d8c3976aa7fc4fb-part1  | SBD device for Cluster 1  |
| Service IP   | 10.137.1.41  | Service IP for Cluster 3  |

# Used Cluster Nodes
A list of cluster nodes (including details) used in this PoC.

|Parameter		|Hostname | Network 						|Note|
|:---------:			|:-----:	 |:-----		|:-----|
| Cluster 1 Node 1		|torappsrv11 | 10.137.0.40 / 255.255.255.240) <br> IMZ VLAN (vlan0@bond0) | SAP DB Sybase HADR Primary |
| Cluster 1 Node 2		|torappsrv12 | 10.137.0.39 / 255.255.255.240) <br> IMZ VLAN (vlan0@bond0) | SAP DB Sybase HADR Stand-by|


**Remember to reflect and adjust these parameters in your specific configuration.**

# Prerequisites

## Cluster
A working SLE HAE cluster (as described in document _SLE-HAE-basic_) has to be setup.

## Middleware
- Sybase HADR is installed and configured on relevant nodes
- automated start of middleware-components during system boot is disabled

# Configure cluster Properties and Resources

The following actions will be executed on one node only (preferable on primary node).

## Set cluster in maintenance mode

```
server:~ # crm configure property maintenance-mode="true"
```
This is to disable all cluster-monitoring activity.

## Define required resources

Configure and commit the required definitions (primitive & group configurations).
Resource Agent script file has to be copied to location /usr/lib/ocf/resource.d/heartbeat
The file should be named as **sybasehadr**:
```
server:~ # crm configure
edit
```
```
# the IP resource
primitive ip_syb IPaddr2 \
        op monitor interval="10s" timeout="20s" \
        params ip="10.137.0.41" \
	meta target-role=Started

# the DB resource, note the additional monitor op with role "Master"
primitive syb_ra sybasehadr \
        params sybhome="/sybase/SA1" syb_ase=ASE-16_0 server=SA1 bkpserver=SA1_BS sybuser=sybsa1 sybpwd=sapHA123 sybadmin=sa dradm=DR_admin drpwd=sapHA123 \
        op start interval=0 timeout=800 \
        op stop interval=0 timeout=240 \
        op promote interval=0 timeout=800 \
        op demote interval=0 timeout=800 \
        op migrate interval=0 timeout=800 \
        op notify interval=0 timeout=600 \
        op monitor interval=20 timeout=60 \
        op monitor interval=22 role=Master timeout=600 \
        meta target-role=Master

# the m/s resource, notifications are required
ms ms_syb_ra syb_ra \
        meta target-role=Started notify=true maintenance=false

#location information
colocation ip_syb_with_master inf: ip_syb:Started ms_syb_ra:Master
order master_after_ip inf: ip_syb:start ms_syb_ra:promote
```

```
commit
exit
server:~ # 
```


## Activate the cluster

Disable the maintenance-mode:


```
server:~ # crm configure property maintenance-mode="false"
```

# Add the Replication Details in Sybase env


Add the RunFile and Replication path under SYBASE environment (/sybase/`<SID>`/SYBASE.sh, /sybase/`<SID>`/SYBASE.csh and /sybase/`<SID>`/SYBASE.env)

Add below Lines in file /sybase/SA1/SYBASE.sh
```
SYBASE_REP="DM/CL1_REP_HADR1"
export SYBASE_REP
REP_RUNFILE="RUN_CL1_REP_HADR1.sh"
export REP_RUNFILE
```

Add below Lines in file /sybase/SA1/SYBASE.csh
```
setenv SYBASE_REP DM/CL1_REP_HADR1
setenv REP_RUNFILE RUN_CL1_REP_HADR1.sh
```

Add below Lines in file /sybase/SA1/SYBASE.env
```
SYBASE_REP=DM/CL1_REP_HADR1
REP_RUNFILE=RUN_CL1_REP_HADR1.sh
```

# Useful commands and information

## Clear SBD device after Node crash

In case a cluster node crashes, the SBD device might be in a incorrect state.
To correct / clear the shared disk, use the following command:

&#x261e; <span style="color:green">**Run on first node**</span>

```
server1:~ # sbd -d "sbd-device" message "cluster-node" clear
```

&#x261d; <span style="color:grey">**Example**</span>

```
torappsrv11:~ # sbd -d /dev/sda1 message torappsrv11 clear
```


# Reference and links

[Fail-Safe Operation of SAP HANA: SUSE Extends Its High-Availability Solution](https://blogs.sap.com/2014/04/04/fail-safe-operation-of-sap-hana-suse-extends-its-high-availability-solution/)  
https://blogs.sap.com/2014/04/04/fail-safe-operation-of-sap-hana-suse-extends-its-high-availability-solution/

[SUSE Documentation: SUSE Linux Enterprise High Availability Extension](https://www.suse.com/documentation/sle_ha/)  
https://www.suse.com/documentation/sle_ha/

[SAP HANA SR Performance Optimized Scenario - SUSE Linux Enterprise Server for SAP Applications 12 SP1](https://www.suse.com/docrep/documents/ir8w88iwu7/suse_linux_enterprise_server_for_sap_applications_12_sp1.pdf)
https://www.suse.com/docrep/documents/ir8w88iwu7/suse_linux_enterprise_server_for_sap_applications_12_sp1.pdf

