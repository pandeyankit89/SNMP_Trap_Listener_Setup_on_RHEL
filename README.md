# SNMP (v2c/v3) Trap Listener Setup on RHEL

- _This guide provides detailed instructions for setting up an SNMP v2 and v3 trap listener on RHEL, printing received traps to the console, and archiving them to a log file. It also includes steps for setting up the listener as a systemd service._

  - [(1) SNMP v2 Trap Listener Setup and Print Received Traps on Console](#1)
  - [(2) SNMP v2 Trap Listener Setup with Log File](#2)
  - [(3) SNMP v2 Trap Listener Setup with `systemctl`](#3)
  - [(4) SNMP v3 Trap Listener Setup with Log File](#4)
---
<a id="1"></a>
## (1) *SNMP v2 Trap Listener Setup and Print Received Traps on Console* :

### Step 1: Log in to RHEL
- Log in to your RHEL system as the `ec2-user`:

```bash
ssh ec2-user@<your-rhel-instance>
```


### Step 2: Upgrade Repository
- Update the system packages:
```bash
sudo dnf upgrade
```

### Step 3: Install Required Packages
- Install :
  - `net-snmp` for `snmptrapd` (trap listener) and
  - `net-snmp-utils` for `snmptrap` (to trigger test traps):
```bash
sudo dnf install -y net-snmp-utils net-snmp
```

- Install `tcpdump` for troubleshooting:
```bash
sudo yum install -y tcpdump
```

### Step 4: Create a Directory for SNMP Traps
- Create a directory to store configuration and log files:
```bash
cd /opt
sudo mkdir snmptraps
```

### Step 5: Create SNMP v2 Configuration File
- Create a configuration file for `snmptrapd`:
```bash
sudo vi /opt/snmptraps/snmptrapd_v2c.conf
```

- Add the following content to the file:
```bash
format2 %V\n% Agent Address: %A \n Agent Hostname: %B \n Date: %H - %J - %K - %L - %M - %Y \n Enterprise OID: %N \n Trap Type: %W \n Trap Sub-Type: %q \n Community/Infosec Context: %P \n Uptime: %T \n Description: %W \n PDU Attribute/Value Pair Array:\n%v \n -------------- \n

authCommunity   log,execute,net public
```

### Step 6: Run the SNMP Trap Listener
- Find the path to `snmptrapd`:
```bash
which snmptrapd
```

- Run the `snmptrapd` listener in the foreground:
```bash
sudo /usr/sbin/snmptrapd -f -C -c /opt/snmptraps/snmptrapd_v2c.conf -Of -Lo udp:162
```

- `-f`: Run in foreground.
- `-C`: Do not read default configuration.
- `-c`: Use the specified configuration file.
- `-Of`: Print OIDs numerically (use `-m ALL` for MIB names if installed).
- `-Lo`: Log to stdout.

### Step 7: Monitor Traffic with `tcpdump`
- Open a new terminal and run `tcpdump` to capture UDP traffic on port 162:
```bash
sudo tcpdump -i any udp port 162 -nn -vv
```

- Expected output:
```bash
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
07:32:47.732132 lo    In  IP (tos 0x0, ttl 64, id 4481, offset 0, flags [DF], proto UDP (17), length 138)
    127.0.0.1.38377 > 127.0.0.1.162: [bad udp cksum 0xfe89 -> 0xdcc7!]  { SNMPv2c { V2Trap(95) R=1372336047  .1.3.6.1.2.1.1.3.0=483067 .1.3.6.1.6.3.1.1.4.1.0=.1.3.6.1.4.1.8072.2.3.0.1 .1.3.6.1.2.1.1.1.0="Test trap from snmptrap" } }
```

### Step 8: Trigger a Test SNMP Trap
- In a new terminal, find the path to `snmptrap`:
```bash
which snmptrap
```

- Trigger a test trap:
```bash
sudo /usr/bin/snmptrap -v 2c -c public localhost '' 1.3.6.1.4.1.8072.2.3.0.1 1.3.6.1.2.1.1.1.0 s "Test trap from snmptrap"
```

- The terminal running `snmptrapd` will display:
```text
NET-SNMP version 5.9.4.pre2
 Agent Address: 0.0.0.0
 Agent Hostname: localhost
 Date: 0 - 1 - 4 - 1 - 1 - 1970
 Enterprise OID: .
 Trap Type: Cold Start
 Trap Sub-Type: 0
 Community/Infosec Context: TRAP2, SNMP v2c, community public
 Uptime: 0
 Description: Cold Start
 PDU Attribute/Value Pair Array:
.iso.org.dod.internet.mgmt.mib-2.system.sysUpTime.sysUpTimeInstance = Timeticks: (592160) 1:38:41.60
.iso.org.dod.internet.snmpV2.snmpModules.snmpMIB.snmpMIBObjects.snmpTrap.snmpTrapOID.0 = OID: .iso.org.dod.internet.private.enterprises.netSnmp.netSnmpExamples.netSnmpExampleNotifications.netSnmpExampleNotificationPrefix.netSnmpExampleHeartbeatNotification
.iso.org.dod.internet.mgmt.mib-2.system.sysDescr.0 = STRING: Test trap from snmptrap
 --------------
```
---
<a id="2"></a>
## (2) *SNMP v2 Trap Listener Setup with Log File :*

### Step 1: Create a Log File
- Create a log file to store traps:
```bash
cd /opt/snmptraps
sudo touch snmptraps.log
sudo chmod 664 snmptraps.log
```

### Step 2: Run the SNMP Trap Listener
- Run `snmptrapd` to log traps to the file:
```bash
sudo /usr/sbin/snmptrapd -Lf /opt/snmptraps/snmptraps.log -c /opt/snmptraps/snmptrapd_v2c.conf udp:162
```

### Step 3: Trigger a Test SNMP Trap
- Trigger a test trap (same as above):
```bash
sudo /usr/bin/snmptrap -v 2c -c public localhost '' 1.3.6.1.4.1.8072.2.3.0.1 1.3.6.1.2.1.1.1.0 s "Test trap from snmptrap"
```

### Step 4: Verify the Log File
- Check the contents of the log file:
```bash
more /opt/snmptraps/snmptraps.log
```

- Expected output:
```text
Cannot bind for clientaddr:  Agent Address: 0.0.0.0
 Agent Hostname: localhost
 Date: 0 - 1 - 4 - 1 - 1 - 1970
 Enterprise OID: .
 Trap Type: Cold Start
 Trap Sub-Type: 0
 Community/Infosec Context: TRAP2, SNMP v2c, community public
 Uptime: 0
 Description: Cold Start
 PDU Attribute/Value Pair Array:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (785104) 2:10:51.04
SNMPv2-MIB::snmpTrapOID.0 = OID: NET-SNMP-EXAMPLES-MIB::netSnmpExampleHeartbeatNotification
SNMPv2-MIB::sysDescr.0 = STRING: Test trap from snmptrap
 --------------
```
---
<a id="3"></a>
## (3) *SNMP v2 Trap Listener Setup with `systemctl` :*

### Step 1: Configure `snmptrapd`
- Edit the default `snmptrapd` configuration file:
```bash
sudo vi /etc/snmp/snmptrapd.conf
```

- Add the following content:
```bash
format2 %V\n% Agent Address: %A \n Agent Hostname: %B \n Date: %H - %J - %K - %L - %M - %Y \n Enterprise OID: %N \n Trap Type: %W \n Trap Sub-Type: %q \n Community/Infosec Context: %P \n Uptime: %T \n Description: %W \n PDU Attribute/Value Pair Array:\n%v \n -------------- \n

authCommunity   log,execute,net public

# Send traps to logfile
[snmp] logOption f /opt/snmptraps/snmptraps.log
```

### Step 2: Disable SELinux (Temporary)
- Temporarily disable SELinux to avoid permission issues:
```bash
sudo setenforce 0
```

### Step 3: Start and Verify `snmptrapd` Service
- Start the `snmptrapd` service:
```bash
sudo systemctl start snmptrapd
```

- Check the service status:
```bash
sudo systemctl status snmptrapd
```

### Step 4: Trigger a Test SNMP Trap
- Trigger a test trap (same as above):
```bash
sudo /usr/bin/snmptrap -v 2c -c public localhost '' 1.3.6.1.4.1.8072.2.3.0.1 1.3.6.1.2.1.1.1.0 s "Test trap from snmptrap"
```

### Step 5: Verify the Log File
- Check the log file for the trap:
```bash
more /opt/snmptraps/snmptraps.log
```

- Expected output (same as above):
```text
Cannot bind for clientaddr:  Agent Address: 0.0.0.0
 Agent Hostname: localhost
 Date: 0 - 1 - 4 - 1 - 1 - 1970
 Enterprise OID: .
 Trap Type: Cold Start
 Trap Sub-Type: 0
 Community/Infosec Context: TRAP2, SNMP v2c, community public
 Uptime: 0
 Description: Cold Start
 PDU Attribute/Value Pair Array:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (785104) 2:10:51.04
SNMPv2-MIB::snmpTrapOID.0 = OID: NET-SNMP-EXAMPLES-MIB::netSnmpExampleHeartbeatNotification
SNMPv2-MIB::sysDescr.0 = STRING: Test trap from snmptrap
 --------------
```
---
<a id="4"></a>
## (4) *SNMP v3 Trap Listener Setup with Log File :*

### Step 1: Create SNMP v3 Configuration File
- Create a configuration file for SNMP v3:
```bash
sudo vi /opt/snmptraps/snmptrapd_v3.conf
```

- Add the following content:
```bash
format2 %V\n% Agent Address: %A \n Agent Hostname: %B \n Date: %H - %J - %K - %L - %M - %Y \n Enterprise OID: %N \n Trap Type: %W \n Trap Sub-Type: %q \n Community/Infosec Context: %P \n Uptime: %T \n Description: %W \n PDU Attribute/Value Pair Array:\n%v \n -------------- \n
createUser -e 0x8000C53Ff64f341c655d11eb8778fa163e914bcc myuser SHA mypassword AES mysecret
authUser log,execute myuser
```

### Step 2: Run the SNMP Trap Listener
- Run `snmptrapd` with the v3 configuration:
```bash
sudo /usr/sbin/snmptrapd -Lf /opt/snmptraps/snmptraps.log -c /opt/snmptraps/snmptrapd_v3.conf udp:162
```

### Step 3: Trigger a Test SNMP v3 Trap
- Trigger a test trap with SNMP v3 parameters:
```bash
sudo /usr/bin/snmptrap -v 3 -e 0x8000C53Ff64f341c655d11eb8778fa163e914bcc -u myuser -a SHA -A mypassword -x AES -X mysecret -l authPriv localhost '' 1.3.6.1.4.1.8072.2.3.0.1 1.3.6.1.2.1.1.1.0 s "Test trap from snmptrap1"
```

### Step 4: Verify the Log File
- Check the log file for the trap:
```bash
more /opt/snmptraps/snmptraps.log
```

- Expected output:
```text
Cannot bind for clientaddr:  Agent Address: 0.0.0.0
 Agent Hostname: localhost
 Date: 0 - 1 - 4 - 1 - 1 - 1970
 Enterprise OID: .
 Trap Type: Cold Start
 Trap Sub-Type: 0
 Community/Infosec Context: TRAP2, SNMP v2c, community public
 Uptime: 0
 Description: Cold Start
 PDU Attribute/Value Pair Array:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (785104) 2:10:51.04
SNMPv2-MIB::snmpTrapOID.0 = OID: NET-SNMP-EXAMPLES-MIB::netSnmpExampleHeartbeatNotification
SNMPv2-MIB::sysDescr.0 = STRING: Test trap from snmptrap
 --------------
```
---
