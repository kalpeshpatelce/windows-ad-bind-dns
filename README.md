# Windows Active Directory with Linux BIND DNS — Complete Integration Guide
End-to-end implementation of Active Directory with external BIND DNS on AWS. Includes SRV record design, TSIG updates, _msdcs zone architecture, and domain join validation.
ReFer the HTML

## Architecture Overview
This setup replaces Windows DNS (which would normally be installed alongside AD DS) with a Linux BIND 9 server as the sole authoritative DNS for the Active Directory domain. All required AD SRV records are pre-populated in BIND before DC promotion. The DC registers its dynamic records into BIND via IP-authenticated DNS updates.
<img width="1069" height="707" alt="image" src="https://github.com/user-attachments/assets/58357404-ae26-477f-afd1-8b81bd86723a" />
### Host & IP Map


| Role | 	Hostname              | IP Address     | 	OS                  | 	Instance  |
| -------| ---------------------|----------------| ---------------------|-------------|
| DNS	   | dns01.corp.local     | 192.168.15.172 | 	Ubuntu 22.04 LTS    | 	t3.small  | 
| DC	   | corp-dc01.corp.local | 192.168.6.34   | 	Windows Server 2022 | 	t3.medium | 
| Client | EC2AMAZ-CL45ERI      | DHCP           | 	Windows Server 2022 | 	t3.small  | 

### Required Ports
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

## 1. AWS Infrastructure Deployment
Local Machine
````BASH
# Install AWS CLI v2 and configure
aws configure
# Enter: Access Key, Secret Key, us-east-1, json

# Verify identity
aws sts get-caller-identity

# Create key pair
aws ec2 create-key-pair \
  --region us-east-1 \
  --key-name ad-lab-keypair \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/ad-lab-keypair.pem
chmod 400 ~/.ssh/ad-lab-keypair.pem

# Get your public IP for SG rules
curl -s ifconfig.me

# Find latest Windows Server 2022 AMI
aws ec2 describe-images \
  --region us-east-1 \
  --owners amazon \
  --filters "Name=name,Values=Windows_Server-2022-English-Full-Base-*" \
            "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].{ID:ImageId,Name:Name}' \
  --output table

# Find latest Ubuntu 22.04 AMI
aws ec2 describe-images \
  --region us-east-1 \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-*" \
            "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].{ID:ImageId,Name:Name}' \
  --output table
````

### Create VPC and Networking
phase1-aws-infrastructure.sh (excerpt)
````BASH
REGION="us-east-1"

# VPC
VPC_ID=$(aws ec2 create-vpc --region "$REGION" \
  --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support

# Private subnet — AD infrastructure
PRIVATE_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)

# Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID"

# NAT Gateway (for private subnet outbound)
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id "$PUBLIC_SUBNET_ID" --allocation-id "$EIP_ALLOC" \
  --query 'NatGateway.NatGatewayId' --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids "$NAT_GW_ID"
````
### Security Groups
````BASH
# DNS Server SG — port 53 from VPC only, SSH from admin IP
DNS_SG_ID=$(aws ec2 create-security-group \
  --group-name "ad-lab-dns-sg" \
  --description "BIND DNS — AD lab" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id "$DNS_SG_ID" \
  --ip-permissions \
  '[{"IpProtocol":"tcp","FromPort":53,"ToPort":53,"IpRanges":[{"CidrIp":"10.0.0.0/16"}]}]'
aws ec2 authorize-security-group-ingress --group-id "$DNS_SG_ID" \
  --ip-permissions \
  '[{"IpProtocol":"udp","FromPort":53,"ToPort":53,"IpRanges":[{"CidrIp":"10.0.0.0/16"}]}]'

# DC SG — all AD ports from VPC, RDP from admin IP
DC_SG_ID=$(aws ec2 create-security-group \
  --group-name "ad-lab-dc-sg" \
  --description "Windows AD DC — AD lab" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)

# AD ports: 88, 389, 445, 464, 3268, 49152-65535
for port in 88 389 445 464 3268 135; do
  aws ec2 authorize-security-group-ingress --group-id "$DC_SG_ID" \
    --ip-permissions \
    "[{\"IpProtocol\":\"tcp\",\"FromPort\":$port,\"ToPort\":$port,\"IpRanges\":[{\"CidrIp\":\"10.0.0.0/16\"}]}]"
done
````

### DHCP Options — Critical AWS Step
````BASH
DHCP_OPTIONS_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configurations \
    "Key=domain-name,Values=corp.local" \
    "Key=domain-name-servers,Values=192.168.15.172" \
    "Key=ntp-servers,Values=169.254.169.123" \
  --query 'DhcpOptions.DhcpOptionsId' --output text)

aws ec2 associate-dhcp-options \
  --vpc-id "$VPC_ID" \
  --dhcp-options-id "$DHCP_OPTIONS_ID"
````

## 2. BIND 9 Installation & Zone Configuration
### System Preparation
dns01
````BASH
# Set hostname
sudo hostnamectl set-hostname dns01.corp.local

# /etc/hosts — FQDN must resolve before DNS starts
sudo tee /etc/hosts > /dev/null << 'EOF'
127.0.0.1       localhost
192.168.15.172  dns01.corp.local dns01
192.168.6.34    corp-dc01.corp.local corp-dc01
EOF

# Disable systemd-resolved (conflicts with BIND on :53)
sudo systemctl disable --now systemd-resolved
sudo rm -f /etc/resolv.conf
sudo tee /etc/resolv.conf > /dev/null << 'EOF'
nameserver 127.0.0.1
nameserver 169.254.169.253
search corp.local
EOF
sudo chattr +i /etc/resolv.conf   # prevent DHCP overwrite
````
### Install BIND + Configure NTP

````BASH
sudo apt-get update -qq
sudo apt-get install -y bind9 bind9utils dnsutils chrony

# NTP via AWS Time Sync (always available on EC2)
sudo tee /etc/chrony/chrony.conf > /dev/null << 'EOF'
server 169.254.169.123 prefer iburst
pool pool.ntp.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
EOF
sudo systemctl enable --now chrony
chronyc tracking | grep -E "Reference|System"

named -v   # verify BIND version
````

named.conf.options
/etc/bind/named.conf.options
````BASH
options {
    directory "/var/cache/bind";

    listen-on { 127.0.0.1; 192.168.15.172; };
    listen-on-v6 { none; };

    allow-query     { localhost; 192.168.0.0/16; };
    allow-recursion { localhost; 192.168.0.0/16; };

    // forward first: uses forwarders, falls back to root hints on failure
    // NOTE: "forward only" causes total DNS failure if upstreams are down
    forwarders { 169.254.169.253; 8.8.8.8; };
    forward first;

    allow-transfer { none; };
    allow-update { none; };   // NEVER set globally — override per-zone only

    version "Not disclosed";
    edns-udp-size 4096;
    max-udp-size  4096;
    dnssec-validation auto;
    auth-nxdomain no;
    recursion yes;
};
````

Generate TSIG Key
````BASH
sudo tsig-keygen -a hmac-sha256 ad-update-key | sudo tee /etc/bind/tsig-ad.key
sudo chown root:bind /etc/bind/tsig-ad.key
sudo chmod 640 /etc/bind/tsig-ad.key
sudo cat /etc/bind/tsig-ad.key   # save this secret value
````

named.conf.local — Zone Declarations
/etc/bind/named.conf.local
````BASH
include "/etc/bind/tsig-ad.key";

acl "ad-dns-updaters" {
    192.168.6.34;   // corp-dc01
};

// _msdcs — MUST be a separate authoritative zone
zone "_msdcs.corp.local" {
    type master;
    file "/etc/bind/zones/db._msdcs.corp.local";
    allow-update  { key "ad-update-key"; 192.168.6.34; 127.0.0.1; };
    allow-query   { 192.168.0.0/16; localhost; };
    allow-transfer { none; };
};

zone "corp.local" {
    type master;
    file "/etc/bind/zones/db.corp.local";
    allow-update  { key "ad-update-key"; 192.168.6.34; 127.0.0.1; };
    allow-query   { 192.168.0.0/16; localhost; };
    allow-transfer { none; };
};

zone "168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168";
    allow-update  { key "ad-update-key"; 192.168.6.34; 127.0.0.1; };
    allow-query   { 192.168.0.0/16; localhost; };
    allow-transfer { none; };
};
````

corp.local Zone File
/etc/bind/zones/db.corp.local
````BASH
$ORIGIN corp.local.
$TTL 3600

@   IN  SOA dns01.corp.local. hostmaster.corp.local. (
                2024010101  ; Serial — increment on every change
                900 300 604800 300 )

@               IN  NS  dns01.corp.local.
; _msdcs delegation — BIND resolves via separate zone in named.conf.local
_msdcs          3600 IN  NS  dns01.corp.local.

; A Records (TTL 3600 — infrastructure changes rarely)
@               3600 IN  A   192.168.6.34
dns01           3600 IN  A   192.168.15.172
corp-dc01       3600 IN  A   192.168.6.34

; Kerberos SRV (TTL 300 — short for fast DC failover)
_kerberos._tcp                  300 IN  SRV  0 100 88  corp-dc01.corp.local.
_kerberos._udp                  300 IN  SRV  0 100 88  corp-dc01.corp.local.
_kerberos._tcp.Default-First-Site-Name._sites  300 IN  SRV  0 100 88  corp-dc01.corp.local.
_kpasswd._tcp                   300 IN  SRV  0 100 464 corp-dc01.corp.local.
_kpasswd._udp                   300 IN  SRV  0 100 464 corp-dc01.corp.local.

; LDAP SRV
_ldap._tcp                      300 IN  SRV  0 100 389 corp-dc01.corp.local.
_ldap._udp                      300 IN  SRV  0 100 389 corp-dc01.corp.local.
_ldap._tcp.Default-First-Site-Name._sites  300 IN  SRV  0 100 389 corp-dc01.corp.local.

; Global Catalog
_gc._tcp                        300 IN  SRV  0 100 3268 corp-dc01.corp.local.
_gc._tcp.Default-First-Site-Name._sites  300 IN  SRV  0 100 3268 corp-dc01.corp.local.

; Forest/Domain partition locators
_ldap._tcp.DomainDNSZones       300 IN  SRV  0 100 389 corp-dc01.corp.local.
_ldap._tcp.ForestDNSZones       300 IN  SRV  0 100 389 corp-dc01.corp.local.
_ldap._tcp.Default-First-Site-Name._sites.DomainDNSZones  300 IN  SRV  0 100 389 corp-dc01.corp.local.
_ldap._tcp.Default-First-Site-Name._sites.ForestDNSZones  300 IN  SRV  0 100 389 corp-dc01.corp.local.

; Kerberos realm TXT (for Linux sssd/realmd clients)
_kerberos                       3600 IN  TXT "CORP.LOCAL"
````

_msdcs Zone File
/etc/bind/zones/db._msdcs.corp.local
````BASH
$ORIGIN _msdcs.corp.local.
$TTL 600

@   IN  SOA dns01.corp.local. hostmaster.corp.local. (
                2024010101
                900 300 604800 300 )

@               IN  NS  dns01.corp.local.

; Global Catalog A record
gc              IN  A   192.168.6.34

; DC GUID CNAME — add after promotion
; Retrieve: (Get-ADDomainController -Identity "CORP-DC01").ObjectGUID
c1771fb0-983c-41dc-9dcb-20ed83363168  IN  CNAME  corp-dc01.corp.local.

; DC role SRV records
_ldap._tcp.dc                   300 IN  SRV  0 100 389  corp-dc01.corp.local.
_ldap._tcp.Default-First-Site-Name._sites.dc  300 IN  SRV  0 100 389  corp-dc01.corp.local.
_ldap._tcp.pdc                  300 IN  SRV  0 100 389  corp-dc01.corp.local.
_ldap._tcp.gc                   300 IN  SRV  0 100 3268 corp-dc01.corp.local.
_ldap._tcp.Default-First-Site-Name._sites.gc  300 IN  SRV  0 100 3268 corp-dc01.corp.local.
_kerberos._tcp.dc               300 IN  SRV  0 100 88   corp-dc01.corp.local.
_kerberos._tcp.Default-First-Site-Name._sites.dc  300 IN  SRV  0 100 88  corp-dc01.corp.local.
````

Set Permissions, Validate & Start
````BASH
sudo mkdir -p /etc/bind/zones /var/log/named
sudo chown -R bind:bind /etc/bind/zones /var/log/named
sudo chmod 755 /etc/bind/zones
sudo chmod 644 /etc/bind/zones/*

sudo named-checkconf
sudo named-checkzone corp.local        /etc/bind/zones/db.corp.local
sudo named-checkzone _msdcs.corp.local /etc/bind/zones/db._msdcs.corp.local

sudo systemctl enable named
sudo systemctl restart named

# Smoke tests
dig @127.0.0.1 corp.local SOA +short
dig @127.0.0.1 _msdcs.corp.local SOA +short    # must have its own SOA
dig @127.0.0.1 _ldap._tcp.corp.local SRV +short
dig @127.0.0.1 _ldap._tcp.dc._msdcs.corp.local SRV +short
````

## 3. AD DS Promotion (Without Windows DNS)
Get Windows Password from AWS
local machine
````BASH
# Wait 4-8 min after first boot, then:
aws ec2 get-password-data \
  --instance-id i-XXXXXXXXXX \
  --priv-launch-key ~/.ssh/ad-lab-keypair.pem \
  --query 'PasswordData' --output text
````
Configure DNS & Install AD DS Role
corp-dc01 · PowerShell Admin

````POWERSHELL
# Point DNS ONLY to Linux BIND
$ifIndex = (Get-NetAdapter | Where-Object Status -eq 'Up').InterfaceIndex
Set-DnsClientServerAddress -InterfaceIndex $ifIndex -ServerAddresses "192.168.15.172"

# Verify BIND is answering
Resolve-DnsName corp.local -Type SOA -Server 192.168.15.172
Resolve-DnsName _ldap._tcp.corp.local -Type SRV -Server 192.168.15.172

# Configure NTP via AWS Time Sync
w32tm /config /manualpeerlist:"169.254.169.123,0x9" /syncfromflags:MANUAL /reliable:YES /update
Restart-Service W32Time; w32tm /resync /force

# Install AD DS role — explicitly WITHOUT DNS role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Verify DNS was NOT installed
(Get-WindowsFeature -Name DNS).Installed   # Must return False
````

Promote to Domain Controller

````POWERSHELL
Import-Module ADDSDeployment

$DSRMPassword = Read-Host -Prompt "DSRM Password" -AsSecureString

Install-ADDSForest `
    -DomainName            "corp.local" `
    -DomainNetbiosName     "CORP" `
    -DomainMode            "WinThreshold" `
    -ForestMode            "WinThreshold" `
    -DatabasePath          "C:\Windows\NTDS" `
    -SysvolPath            "C:\Windows\SYSVOL" `
    -LogPath               "C:\Windows\NTDS" `
    -InstallDns            $false `
    -CreateDnsDelegation   $false `
    -NoRebootOnCompletion  $false `
    -SafeModeAdministratorPassword $DSRMPassword `
    -Force:$true
````

### 3B. Post-Promotion Verification
****
````POWRESHELL
# Re-assert DNS (promotion reboot sometimes resets it)
$ifIndex = (Get-NetAdapter | Where-Object Status -eq 'Up').InterfaceIndex
Set-DnsClientServerAddress -InterfaceIndex $ifIndex -ServerAddresses "192.168.15.172"
Clear-DnsClientCache

# Force DNS record registration
ipconfig /registerdns
Restart-Service Netlogon -Force

# ── RETRIEVE DC GUID — paste into phase4 on Linux ──
$dcGuid = (Get-ADDomainController -Identity $env:COMPUTERNAME).ObjectGUID
Write-Host "DC GUID: $($dcGuid.ToString().ToLower())" -ForegroundColor Yellow
$dcGuid.ToString().ToLower() | Out-File "C:\dc-guid.txt"

# Validate domain health
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, PDCEmulator
nltest /dsgetdc:corp.local

# Verify shares
net share   # Must show SYSVOL and NETLOGON

# Full DNS diagnostic
dcdiag /test:DNS /DnsBasic /DnsDynamicUpdate /v
````
## 4. Post-Promotion DNS Update
dns01
````POWERSHELL
MSDCS_ZONE="/etc/bind/zones/db._msdcs.corp.local"
DC_GUID="c1771fb0-983c-41dc-9dcb-20ed83363168"   # from phase3b

# Remove old GUID line (idempotent)
sudo sed -i "/${DC_GUID}/d" "$MSDCS_ZONE"

# Increment serial
SERIAL=$(grep -oP '\d{10}' "$MSDCS_ZONE" | head -1)
sudo sed -i "s/$SERIAL/$((SERIAL+1))/" "$MSDCS_ZONE"

# Append GUID CNAME
echo "${DC_GUID}  IN  CNAME  corp-dc01.corp.local." | sudo tee -a "$MSDCS_ZONE"

sudo named-checkzone _msdcs.corp.local "$MSDCS_ZONE"
sudo rndc reload
sleep 2

# Verify
dig @127.0.0.1 "${DC_GUID}._msdcs.corp.local" CNAME +short
# Expected: corp-dc01.corp.local.
````

Full DNS Validation
````BASH
# corp.local zone
dig @127.0.0.1 corp.local SOA +short
dig @127.0.0.1 corp-dc01.corp.local A +short
dig @127.0.0.1 _ldap._tcp.corp.local SRV +short
dig @127.0.0.1 _kerberos._tcp.corp.local SRV +short
dig @127.0.0.1 _gc._tcp.corp.local SRV +short

# _msdcs zone — must have its own SOA
dig @127.0.0.1 _msdcs.corp.local SOA +short
dig @127.0.0.1 _ldap._tcp.dc._msdcs.corp.local SRV +short
dig @127.0.0.1 _ldap._tcp.pdc._msdcs.corp.local SRV +short
dig @127.0.0.1 gc._msdcs.corp.local A +short

# Reverse lookup
dig @127.0.0.1 -x 192.168.6.34 +short

# Dynamic update test
sudo nsupdate -k /etc/bind/tsig-ad.key << 'EOF'
server 127.0.0.1
zone corp.local.
update add test-ddns.corp.local. 300 A 192.168.6.99
send
EOF
dig @127.0.0.1 test-ddns.corp.local A +short   # expect 192.168.6.99
````

## 5. Client Domain Join
client01
````POWERSHELL
# Step 1: Point DNS to BIND
$ifIndex = (Get-NetAdapter | Where-Object Status -eq 'Up').InterfaceIndex
Set-DnsClientServerAddress -InterfaceIndex $ifIndex -ServerAddresses "192.168.15.172"
Clear-DnsClientCache

# Step 2: Pre-flight DNS check
Resolve-DnsName corp.local -Type SOA -Server 192.168.15.172
Resolve-DnsName _ldap._tcp.corp.local -Type SRV -Server 192.168.15.172
Resolve-DnsName _ldap._tcp.dc._msdcs.corp.local -Type SRV -Server 192.168.15.172
Resolve-DnsName _ldap._tcp.pdc._msdcs.corp.local -Type SRV -Server 192.168.15.172

# Step 3: Port connectivity
foreach ($port in @(88, 389, 445, 464, 3268)) {
    $t = Test-NetConnection -ComputerName 192.168.6.34 -Port $port -WarningAction SilentlyContinue
    Write-Host "Port $port : $($t.TcpTestSucceeded)"
}

# Step 4: Time sync
w32tm /config /manualpeerlist:"169.254.169.123,0x9" /syncfromflags:MANUAL /reliable:YES /update
Restart-Service W32Time; w32tm /resync /force

# Step 5: JOIN — note: no -OUPath (CN=Computers is a container, not an OU)
Add-Computer `
    -DomainName "corp.local" `
    -Credential (Get-Credential "CORP\Administrator") `
    -Force `
    -Restart
````

Post-Join Validation
client01 · after reboot · login as CORP\Administrator
````POWERSHELL
(Get-WmiObject Win32_ComputerSystem).Domain          # corp.local
whoami                                                # CORP\Administrator
klist                                                 # Kerberos tickets
Get-ADComputer -Filter * | Select-Object Name, DistinguishedName
Get-ADUser -Filter * | Select-Object SamAccountName
````

### FIX AppArmor Blocks Journal File Creation
🔥 Symptom: journal open failed: permission denied
BIND accepts the dynamic update, processes it, then fails to write the .jnl journal file. The record is discarded. The update appears to be "accepted" in syslog but never persists.

Syslog shows the real cause:

````log
/etc/bind/zones/db.corp.local.jnl: create: permission denied
apparmor="DENIED" operation="mknod" profile="named"
name="/etc/bind/zones/db.corp.local.jnl"
requested_mask="c" denied_mask="c"
````
AppArmor's named profile does not allow file creation in /etc/bind/zones/. chown and chmod have no effect — AppArmor blocks at the kernel level regardless of filesystem permissions.

### Fix — AppArmor Local Override
dns01
```bash
# Add write permission for zones directory to the named AppArmor profile
sudo tee /etc/apparmor.d/local/usr.sbin.named > /dev/null << 'EOF'
/etc/bind/zones/ rw,
/etc/bind/zones/** rw,
EOF

# Reload the profile
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named

# Verify
sudo aa-status | grep named

# Test — journal file must now be created
sudo nsupdate -k /etc/bind/tsig-ad.key << 'EOF'
server 127.0.0.1
zone corp.local.
update add test-ddns.corp.local. 300 A 192.168.6.99
send
EOF
ls -la /etc/bind/zones/db.corp.local.jnl   # Must now exist
````
### Dynamic Update REFUSED
Fix — Correct allow-update
```bash
# Each zone must list BOTH the TSIG key AND the DC IP AND 127.0.0.1
sudo sed -i \
  's/allow-update  { key "ad-update-key"; };/allow-update { key "ad-update-key"; 192.168.6.34; 127.0.0.1; };/g' \
  /etc/bind/named.conf.local

sudo named-checkconf && sudo rndc reload
````

Correct nsupdate Usage
```bash
# CORRECT — with TSIG key (authenticated)
sudo nsupdate -k /etc/bind/tsig-ad.key << 'EOF'
server 127.0.0.1
zone corp.local.
update add test.corp.local. 300 A 192.168.6.99
send
EOF

# WRONG — no key, gets REFUSED unless 127.0.0.1 is in allow-update
sudo nsupdate << 'EOF'
server 127.0.0.1
...
EOF
````

### FIX _msdcs Zone Architecture
🔥 Wrong Design: _msdcs as a subdomain delegation inside corp.local
bind
❌ Wrong
; Inside db.corp.local — WRONG
````BASH
_msdcs    IN  NS  dns01.corp.local.   ; no separate zone declared
````
bind
✅ Correct — named.conf.local
; Separate authoritative zone declaration
````BIND
zone "_msdcs.corp.local" {
    type master;
    file "/etc/bind/zones/db._msdcs.corp.local";
    allow-update { key "ad-update-key"; 192.168.6.34; 127.0.0.1; };
};
````
AD's DsGetDcName() queries _msdcs.corp.local as an independent authoritative zone. If BIND has no zone declaration for it, SOA lookups return SERVFAIL and DC Locator fails silently across the entire domain.


### FIX Domain Join: "System Cannot Find the File"
🔥 Error: NetpProvisionComputerAccount: LDAP creation failed: 0x2
NetSetup.log reveals:

```log
NetpGetComputerObjectDn: Specified path 'CN=Computers,DC=corp,DC=local' is not an OU
NetpProvisionComputerAccount: Cannot retry downlevel, specifying OU is not supported
````
CN=Computers is a built-in Container, not an OU. When passed to -OUPath, Windows refuses it. The fix is to remove -OUPath entirely — Windows places the computer in CN=Computers automatically.

### Correct Join Command
✅ Correct — no -OUPath
````POWERSHELL
Add-Computer `
    -DomainName "corp.local" `
    -Credential (Get-Credential "CORP\Administrator") `
    -Force `
    -Restart
````


## REFERENCE Complete AD DNS Record Reference

| Record                      | 	Zone              | 	Type      | 	TTL | 	Purpose               |
|-----------------------------|---------------------|-------------|--------|------------------------|
| @                           | 	corp.local        | 	SOA/NS/A  | 	3600 |	Domain root           |
| corp-dc01                   | 	corp.local        | 	A	        |   3600 |	DC address            |
| _kerberos._tcp              | 	corp.local        | 	SRV :88   | 	300  |	KDC (TCP)             |
| _kerberos._udp              | 	corp.local        | 	SRV :88   | 	300  |	KDC (UDP)             | 
| _kpasswd._tcp/udp           | 	corp.local        | 	SRV :464  | 	300  |	Password change       |
| _ldap._tcp                  | 	corp.local        | 	SRV :389  | 	300  |	LDAP (TCP)            |
| _ldap._udp                  | 	corp.local        | 	SRV :389  | 	300  |	LDAP (UDP)            |
| _gc._tcp                    | 	corp.local        | 	SRV :3268 | 	300  |	Global Catalog        |
| _ldap._tcp.*.DomainDNSZones | 	corp.local        | 	SRV       | 	300  |	Domain partition      |
| _ldap._tcp.*.ForestDNSZones | 	corp.local        | 	SRV       | 	300	 |  Forest partition      |
| _kerberos                   | 	corp.local        | 	TXT       | 	3600 |  Realm (Linux clients) |
| @                           | 	_msdcs.corp.local | 	SOA/NS    | 	600  |	_msdcs authority      |
| gc                          | 	_msdcs.corp.local | 	A	        |   600  |	GC IP                 |
| _ldap._tcp.dc               | 	_msdcs.corp.local | 	SRV :389  | 	300	 |  Any DC                |
| _ldap._tcp.pdc              | 	_msdcs.corp.local | 	SRV :389  | 	300	 |  PDC Emulator          |
| _ldap._tcp.gc               | 	_msdcs.corp.local | 	SRV :3268 | 	300	 |  GC in _msdcs          |
| _kerberos._tcp.dc           | 	_msdcs.corp.local | 	SRV :88   | 	300	 |  KDC in _msdcs         |
| <DC-GUID>                   | 	_msdcs.corp.local | 	CNAME	    |   600	 |  DC by GUID            |
All _sites variants (e.g., _ldap._tcp.Default-First-Site-Name._sites) must also be present in both zones for site-aware DC locator to function.

## Troubleshooting Checklist
Quick Diagnostics — Linux DNS Server
````BASH
# Config and zone syntax
sudo named-checkconf
sudo named-checkzone corp.local        /etc/bind/zones/db.corp.local
sudo named-checkzone _msdcs.corp.local /etc/bind/zones/db._msdcs.corp.local

# Is BIND listening on the right IP?
ss -tulpn | grep ":53"

# AppArmor denials?
sudo journalctl -u apparmor --no-pager -n 20 | grep named

# Dynamic update log
sudo journalctl -u named --no-pager -n 30

# Test dynamic update
sudo nsupdate -k /etc/bind/tsig-ad.key << 'EOF'
server 127.0.0.1
zone corp.local.
update add debug.corp.local. 60 A 192.168.6.100
send
EOF
dig @127.0.0.1 debug.corp.local A +short
````

Quick Diagnostics — Windows DC
````POWERSHELL
# DC Locator — most important single test
nltest /dsgetdc:corp.local /force
nltest /dsgetdc:corp.local /pdc /force

# Authoritative DNS diagnostic
dcdiag /test:DNS /DnsBasic /DnsDynamicUpdate /v

# Kerberos
klist; klist tgt

# Is SysvolReady = 1?
(Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters").SysvolReady

# SYSVOL/NETLOGON shares exist?
net share | Select-String "SYSVOL|NETLOGON"

# NetSetup.log — exact join error
Get-Content C:\Windows\debug\NetSetup.LOG | Select-Object -Last 40
````
