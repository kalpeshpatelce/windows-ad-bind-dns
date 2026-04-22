# Windows Active Directory with Linux BIND DNS — Complete Integration Guide
End-to-end implementation of Active Directory with external BIND DNS on AWS. Includes SRV record design, TSIG updates, _msdcs zone architecture, and domain join validation.
ReFer the HTML

# Architecture Overview
This setup replaces Windows DNS (which would normally be installed alongside AD DS) with a Linux BIND 9 server as the sole authoritative DNS for the Active Directory domain. All required AD SRV records are pre-populated in BIND before DC promotion. The DC registers its dynamic records into BIND via IP-authenticated DNS updates.
<img width="1069" height="707" alt="image" src="https://github.com/user-attachments/assets/58357404-ae26-477f-afd1-8b81bd86723a" />
# Host & IP Map


| Role | 	Hostname              | IP Address     | 	OS                  | 	Instance  |
| -------| ---------------------|----------------| ---------------------|-------------|
| DNS	   | dns01.corp.local     | 192.168.15.172 | 	Ubuntu 22.04 LTS    | 	t3.small  | 
| DC	   | corp-dc01.corp.local | 192.168.6.34   | 	Windows Server 2022 | 	t3.medium | 
| Client | EC2AMAZ-CL45ERI      | DHCP           | 	Windows Server 2022 | 	t3.small  | 

# Required Ports
| Port        | 	Protocol|	Service             | 	Direction              | 
|-------------|-----------|---------------------|--------------------------|
| 53          | 	TCP/UDP |	DNS                 | 	All hosts ↔ DNS server | 
| 88          | 	TCP/UDP |	Kerberos            | 	Clients ↔ DC           | 
| 135         | 	TCP     |	RPC Endpoint Mapper | 	Clients ↔ DC           | 
| 389         | 	TCP/UDP |	LDAP                | 	Clients ↔ DC           | 
| 445         | 	TCP     |	SMB / SYSVOL        | 	Clients ↔ DC           | 
| 464         | 	TCP/UDP |	Kerberos kpasswd    | 	Clients ↔ DC           | 
| 636         | 	TCP     |	LDAPS               | 	Clients ↔ DC           | 
| 3268        | 	TCP     |	Global Catalog      | 	Clients ↔ DC           | 
| 49152–65535 | 	TCP	    | RPC Dynamic         | 	DC ↔ DC                | 
| 3389        | 	TCP	    | RDP (admin)         | 	Admin IP → DC/Client   | 
| 22          | 	TCP	    | SSH (admin)         | 	Admin IP → DNS server  | 
