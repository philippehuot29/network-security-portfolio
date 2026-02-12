# Lab 5: High Availability Network with HSRP

## 🎯 Objective
Implement First Hop Redundancy Protocol (HSRP) to provide gateway redundancy and eliminate single points of failure. Configure automatic failover between routers to ensure continuous network availability even during router failures.

**Real-World Scenario:** Critical business applications require 99.99% uptime (4 minutes downtime per month). Single gateway router = single point of failure. HSRP provides redundant gateways with sub-second failover, ensuring business continuity.

## 🔧 Equipment Used
- 3 Cisco Routers (R1-Primary, R2-Standby, R3-ISP)
- 2 Cisco Switches (SW1-Core, SW2-Access)
- 4 PCs (testing workstations)
- Software: Cisco Packet Tracer 8.2+ or GNS3

## 📋 Network Design

### Network Topology
```
                    [INTERNET]
                        |
                    [R3 - ISP]
                        |
              __________|__________
             |                     |
         [R1 - Primary]        [R2 - Standby]
         Priority: 110         Priority: 100 (default)
             |                     |
             |_____________________|
                        |
                   [SW1 - Core]
                        |
                   [SW2 - Access]
                        |
              __________|__________
             |          |          |
            PC1        PC2        PC3

Virtual IP (HSRP): 192.168.1.1
Physical R1: 192.168.1.2
Physical R2: 192.168.1.3
```

### HSRP Concept Visualization
```
Normal Operation:
PC1 → Virtual IP (192.168.1.1) → R1 (ACTIVE) → Internet
                                  R2 (STANDBY) waits

After R1 Failure:
PC1 → Virtual IP (192.168.1.1) → R2 (ACTIVE) → Internet
                                  R1 (DOWN)

Failover Time: < 3 seconds
```

### IP Addressing Scheme

**LAN Segment (192.168.1.0/24):**
| Device | Interface | IP Address | Role |
|--------|-----------|------------|------|
| R1 | Gig0/0 (LAN) | 192.168.1.2/24 | HSRP Active (Priority 110) |
| R2 | Gig0/0 (LAN) | 192.168.1.3/24 | HSRP Standby (Priority 100) |
| HSRP VIP | N/A | 192.168.1.1/24 | Virtual Gateway (floats) |
| PC1 | NIC | 192.168.1.10/24 | End user |
| PC2 | NIC | 192.168.1.11/24 | End user |
| PC3 | NIC | 192.168.1.12/24 | End user |

**WAN Segment (203.0.113.0/29):**
| Device | Interface | IP Address | Role |
|--------|-----------|------------|------|
| R1 | Gig0/1 (WAN) | 203.0.113.2/29 | Primary uplink |
| R2 | Gig0/1 (WAN) | 203.0.113.3/29 | Standby uplink |
| R3 (ISP) | Gig0/0 | 203.0.113.1/29 | Internet gateway |

### HSRP Configuration Plan

**HSRP Group 1 (LAN):**
- Virtual IP: 192.168.1.1
- R1 Priority: 110 (higher = preferred active)
- R2 Priority: 100 (default)
- Preemption: Enabled (R1 reclaims active after recovery)
- Timers: Hello 3s, Hold 10s
- Authentication: MD5 (security)
- Interface Tracking: Monitor WAN links (failover if internet down)

**Expected Behavior:**
1. **Normal:** R1 is ACTIVE, R2 is STANDBY
2. **R1 fails:** R2 becomes ACTIVE (3 second failover)
3. **R1 recovers:** R1 preempts and becomes ACTIVE again
4. **R1 internet fails:** R1 priority decreases, R2 becomes ACTIVE

---

## ⚙️ Configuration Steps

### PHASE 1: BASIC ROUTER AND SWITCH CONFIGURATION

#### Step 1: Configure R1 (Primary HSRP Router)
```cisco
Router> enable
Router# configure terminal
Router(config)# hostname R1

! Configure LAN interface
R1(config)# interface gig0/0
R1(config-if)# description LAN Interface - HSRP Primary
R1(config-if)# ip address 192.168.1.2 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

! Configure WAN interface
R1(config)# interface gig0/1
R1(config-if)# description WAN to ISP
R1(config-if)# ip address 203.0.113.2 255.255.255.248
R1(config-if)# no shutdown
R1(config-if)# exit

! Configure default route to ISP
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

#### Step 2: Configure R2 (Standby HSRP Router)
```cisco
Router> enable
Router# configure terminal
Router(config)# hostname R2

! Configure LAN interface
R2(config)# interface gig0/0
R2(config-if)# description LAN Interface - HSRP Standby
R2(config-if)# ip address 192.168.1.3 255.255.255.0
R2(config-if)# no shutdown
R2(config-if)# exit

! Configure WAN interface
R2(config)# interface gig0/1
R2(config-if)# description WAN to ISP
R2(config-if)# ip address 203.0.113.3 255.255.255.248
R2(config-if)# no shutdown
R2(config-if)# exit

! Configure default route to ISP
R2(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

#### Step 3: Configure R3 (ISP Router - Simulation)
```cisco
Router> enable
Router# configure terminal
Router(config)# hostname R3-ISP

! Configure interface to R1
R3-ISP(config)# interface gig0/0
R3-ISP(config-if)# description WAN Segment
R3-ISP(config-if)# ip address 203.0.113.1 255.255.255.248
R3-ISP(config-if)# no shutdown
R3-ISP(config-if)# exit

! Configure routes back to LAN
R3-ISP(config)# ip route 192.168.1.0 255.255.255.0 203.0.113.2
R3-ISP(config)# ip route 192.168.1.0 255.255.255.0 203.0.113.3 10
! Second route has higher AD (10) = backup path
```

#### Step 4: Configure Switches
```cisco
! On SW1 (Core Switch)
Switch> enable
Switch# configure terminal
Switch(config)# hostname SW1-Core

! Create VLAN 1 interface (default)
SW1-Core(config)# interface vlan 1
SW1-Core(config-if)# ip address 192.168.1.100 255.255.255.0
SW1-Core(config-if)# no shutdown
SW1-Core(config-if)# exit

! Set default gateway to HSRP VIP
SW1-Core(config)# ip default-gateway 192.168.1.1

! On SW2 (Access Switch) - similar configuration
Switch(config)# hostname SW2-Access
SW2-Access(config)# interface vlan 1
SW2-Access(config-if)# ip address 192.168.1.101 255.255.255.0
SW2-Access(config-if)# no shutdown
SW2-Access(config-if)# exit
SW2-Access(config)# ip default-gateway 192.168.1.1
```

#### Step 5: Verify Basic Connectivity (Before HSRP)

**Test from R1 to R2:**
```cisco
R1# ping 192.168.1.3
!!!!!
Success rate is 100 percent (5/5)
```

**Test from R1 to ISP:**
```cisco
R1# ping 203.0.113.1
!!!!!
Success rate is 100 percent (5/5)
```

**Configure PCs with default gateway:**
- IP: 192.168.1.10-12
- Subnet: 255.255.255.0
- Gateway: 192.168.1.1 (HSRP VIP - not yet configured)

---

### PHASE 2: BASIC HSRP CONFIGURATION

#### Step 6: Configure HSRP on R1 (Primary)
```cisco
R1(config)# interface gig0/0
R1(config-if)# standby version 2
R1(config-if)# standby 1 ip 192.168.1.1
R1(config-if)# standby 1 priority 110
R1(config-if)# standby 1 preempt
R1(config-if)# exit
```

**Command Breakdown:**
- **standby version 2:** Use HSRPv2 (supports IPv6, more groups)
- **standby 1 ip 192.168.1.1:** Create HSRP group 1 with virtual IP
- **standby 1 priority 110:** Higher than default (100) = preferred active
- **standby 1 preempt:** Take back active role when priority is higher

#### Step 7: Configure HSRP on R2 (Standby)
```cisco
R2(config)# interface gig0/0
R2(config-if)# standby version 2
R2(config-if)# standby 1 ip 192.168.1.1
R2(config-if)# standby 1 priority 100
R2(config-if)# standby 1 preempt
R2(config-if)# exit
```

**Note:** Priority 100 is default, explicitly set for clarity

#### Step 8: Verify HSRP Status

**On R1 (should be ACTIVE):**
```cisco
R1# show standby brief

Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           192.168.1.3     192.168.1.1

✅ R1 is ACTIVE (Priority 110 > 100)
```

**On R2 (should be STANDBY):**
```cisco
R2# show standby brief

Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    100 P Standby  192.168.1.2     local           192.168.1.1

✅ R2 is STANDBY
```

**Detailed HSRP information:**
```cisco
R1# show standby

GigabitEthernet0/0 - Group 1 (version 2)
  State is Active
    2 state changes, last state change 00:03:24
  Virtual IP address is 192.168.1.1
  Active virtual MAC address is 0000.0C9F.F001
    Local virtual MAC address is 0000.0C9F.F001 (v2 default)
  Hello time 3 sec, hold time 10 sec
    Next hello sent in 1.328 secs
  Preemption enabled
  Active router is local
  Standby router is 192.168.1.3, priority 100 (expires in 9.184 sec)
  Priority 110 (configured 110)
  Group name is "hsrp-Gi0/0-1" (default)
```

**Key Information:**
- **State: Active** - R1 is forwarding traffic
- **Virtual MAC: 0000.0C9F.F001** - HSRP virtual MAC address
- **Standby router: 192.168.1.3** - R2 is backup
- **Preemption enabled** - R1 will reclaim active role

---

### PHASE 3: HSRP OPTIMIZATION AND SECURITY

#### Step 9: Configure HSRP Timers (Faster Failover)
```cisco
! On R1
R1(config)# interface gig0/0
R1(config-if)# standby 1 timers 1 3
R1(config-if)# exit

! On R2
R2(config)# interface gig0/0
R2(config-if)# standby 1 timers 1 3
R2(config-if)# exit
```

**Timer Values:**
- **Hello: 1 second** (send hello every 1 sec)
- **Hold: 3 seconds** (declare peer dead after 3 sec no hello)
- **Failover time:** ~3 seconds (previously 10 seconds)

**Trade-off:** Faster failover vs. network stability (too aggressive = flapping)

---

#### Step 10: Configure HSRP Authentication (Security)
```cisco
! On R1
R1(config)# interface gig0/0
R1(config-if)# standby 1 authentication md5 key-string HSRP$ecur3!
R1(config-if)# exit

! On R2
R2(config)# interface gig0/0
R2(config-if)# standby 1 authentication md5 key-string HSRP$ecur3!
R2(config-if)# exit
```

**Why authenticate?**
- Prevents rogue router from injecting itself as active gateway
- MD5 hash verifies HSRP packets from legitimate peers
- Required for secure environments (PCI-DSS, HIPAA)

**Verify authentication:**
```cisco
R1# show standby | include Authentication
  Authentication MD5, key-string "HSRP$ecur3!"
```

---

#### Step 11: Configure Interface Tracking (Intelligent Failover)
```cisco
! On R1 - Track WAN interface
R1(config)# track 1 interface gig0/1 line-protocol

R1(config)# interface gig0/0
R1(config-if)# standby 1 track 1 decrement 20
R1(config-if)# exit

! On R2 - Track WAN interface
R2(config)# track 1 interface gig0/1 line-protocol

R2(config)# interface gig0/0
R2(config-if)# standby 1 track 1 decrement 20
R2(config-if)# exit
```

**How tracking works:**
1. **Normal:** R1 priority = 110, R2 priority = 100
2. **R1 WAN fails:** R1 priority = 110 - 20 = 90 (lower than R2)
3. **HSRP failover:** R2 becomes ACTIVE (priority 100 > 90)
4. **Result:** Traffic flows through R2's working WAN link

**Verify tracking:**
```cisco
R1# show track 1
Track 1
  Interface GigabitEthernet0/1 line-protocol
  Line protocol is Up
    1 change, last change 00:04:23
  Tracked by:
    HSRP GigabitEthernet0/0 1
```

---

#### Step 12: Configure HSRP Load Balancing (Optional - Advanced)

**Scenario:** Two HSRP groups for load distribution
```cisco
! On R1 - Active for Group 1, Standby for Group 2
R1(config)# interface gig0/0
R1(config-if)# standby 1 ip 192.168.1.1
R1(config-if)# standby 1 priority 110
R1(config-if)# standby 1 preempt

R1(config-if)# standby 2 ip 192.168.1.254
R1(config-if)# standby 2 priority 100
R1(config-if)# standby 2 preempt
R1(config-if)# exit

! On R2 - Standby for Group 1, Active for Group 2
R2(config)# interface gig0/0
R2(config-if)# standby 1 ip 192.168.1.1
R2(config-if)# standby 1 priority 100
R2(config-if)# standby 1 preempt

R2(config-if)# standby 2 ip 192.168.1.254
R2(config-if)# standby 2 priority 110
R2(config-if)# standby 2 preempt
R2(config-if)# exit
```

**Load balancing:**
- Half of PCs use gateway 192.168.1.1 (R1 active)
- Half of PCs use gateway 192.168.1.254 (R2 active)
- Distributes traffic across both routers

**Verify both groups:**
```cisco
R1# show standby brief

Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           192.168.1.3     192.168.1.1
Gi0/0       2    100 P Standby  192.168.1.3     local           192.168.1.254
```

---

## ✅ Testing & Verification

### Test 1: Normal Operation (R1 Active, R2 Standby)

**From PC1, check default gateway:**
```
C:\> ipconfig

IP Address: 192.168.1.10
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.1.1

C:\> ping 192.168.1.1
Reply from 192.168.1.1: bytes=32 time<1ms TTL=255

✅ SUCCESS - Virtual IP reachable
```

**Verify which router is active:**
```
C:\> arp -a

Interface: 192.168.1.10
  Internet Address      Physical Address      Type
  192.168.1.1           00-00-0c-9f-f0-01     dynamic  ← HSRP virtual MAC
  192.168.1.2           0001.9641.2D01        dynamic  ← R1 physical MAC
  192.168.1.3           0002.1734.AB02        dynamic  ← R2 physical MAC
```

**On R1, verify state:**
```cisco
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           192.168.1.3     192.168.1.1

✅ R1 is ACTIVE, R2 is STANDBY
```

---

### Test 2: Router Failure (R1 Goes Down)

**Simulate R1 failure - shutdown LAN interface:**
```cisco
R1(config)# interface gig0/0
R1(config-if)# shutdown
```

**On R2, observe transition:**
```cisco
R2# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    100 P Active   local           unknown         192.168.1.1
                      ^^^^^^
                    R2 is now ACTIVE!
```

**Test from PC1 (continuous ping during failover):**
```
C:\> ping 192.168.1.1 -t

Reply from 192.168.1.1: bytes=32 time=2ms TTL=255
Reply from 192.168.1.1: bytes=32 time=1ms TTL=255
Request timed out.  ← Failover happening (~1-2 lost packets)
Request timed out.
Reply from 192.168.1.1: bytes=32 time=3ms TTL=255
Reply from 192.168.1.1: bytes=32 time=2ms TTL=255

✅ SUCCESS - Failover completed in ~3 seconds
```

**Verify internet connectivity still works:**
```
C:\> ping 8.8.8.8
Reply from 8.8.8.8: bytes=32 time=15ms

✅ Traffic now flowing through R2
```

---

### Test 3: Router Recovery with Preemption

**Restore R1:**
```cisco
R1(config)# interface gig0/0
R1(config-if)# no shutdown
```

**Wait 5-10 seconds, check HSRP state:**
```cisco
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           192.168.1.3     192.168.1.1
                      ^^^^^^
                  R1 reclaimed ACTIVE role (preemption)
```

**On R2:**
```cisco
R2# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    100 P Standby  192.168.1.2     local           192.168.1.1
                      ^^^^^^^
                  R2 back to STANDBY
```

**From PC1 (continuous ping):**
```
C:\> ping 192.168.1.1 -t

Reply from 192.168.1.1: bytes=32 time=2ms
Request timed out.  ← Brief interruption during preemption
Reply from 192.168.1.1: bytes=32 time=1ms

✅ R1 successfully preempted and became active
```

---

### Test 4: WAN Link Failure (Interface Tracking)

**Simulate R1 WAN link failure:**
```cisco
R1(config)# interface gig0/1
R1(config-if)# shutdown
```

**Check tracking status:**
```cisco
R1# show track 1
Track 1
  Interface GigabitEthernet0/1 line-protocol
  Line protocol is Down  ← WAN link down
    2 changes, last change 00:00:05
  Tracked by:
    HSRP GigabitEthernet0/0 1
```

**Check HSRP priority:**
```cisco
R1# show standby | include Priority
  Priority 90 (configured 110)
          ^^
      Decremented by 20 due to tracking!
```

**R1 priority now 90 < R2 priority 100, so failover:**
```cisco
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    90  P Standby  192.168.1.3     local           192.168.1.1
                 ^^              ^^^^^^^^^^^
              Lower priority   R2 is now active
```

**On R2:**
```cisco
R2# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    100 P Active   local           192.168.1.2     192.168.1.1

✅ R2 became active because R1's WAN link failed
```

**Test from PC1:**
```
C:\> tracert 8.8.8.8

1    <1 ms     192.168.1.1 (HSRP VIP)
2     2 ms     203.0.113.3 (R2's WAN)  ← Now using R2's internet!
3     5 ms     8.8.8.8

✅ Traffic automatically rerouted through working WAN link
```

---

### Test 5: HSRP State Change Monitoring

**Enable HSRP debugging (before triggering failover):**
```cisco
R1# debug standby events

! Then shutdown R1's interface
R1(config)# interface gig0/0
R1(config-if)# shutdown

! Debug output:
HSRP: Gi0/0 Grp 1 Active: d/Disabled (interface down)
HSRP: Gi0/0 Grp 1 Active -> Init
HSRP: Gi0/0 Grp 1 Removing Active router
```

**On R2 (simultaneously):**
```cisco
R2# debug standby events

! Debug output:
HSRP: Gi0/0 Grp 1 Standby: c/Active timer expired (192.168.1.2)
HSRP: Gi0/0 Grp 1 Standby -> Active
HSRP: Gi0/0 Grp 1 Active: Announce (3) sent
HSRP: Gi0/0 Grp 1 Now active
```

**Disable debugging:**
```cisco
R1# undebug all
R2# undebug all
```

---

### Test 6: Split-Brain Scenario (Monitoring Only)

**Scenario:** What if R1 and R2 can't communicate?

**Simulate:** Disconnect switch link between R1 and R2 (in real network)

**Result:**
- Both routers think peer is down
- Both become ACTIVE (split-brain)
- PCs connected to SW1 use R1
- PCs connected to SW2 use R2
- **Problem:** Duplicate IP addresses, routing inconsistency

**Detection:**
```cisco
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           unknown         192.168.1.1

R2# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    100 P Active   local           unknown         192.168.1.1
                      ^^^^^^                    ^^^^^^^
                  Both ACTIVE!              No standby seen!
```

**Prevention:** Redundant switch infrastructure (this lab has it)

---

## 🐛 Troubleshooting Issues I Encountered

### Problem 1: HSRP neighbors not forming

**Symptom:**
```cisco
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Init     unknown         unknown         192.168.1.1
                      ^^^^
                   Stuck in Init state!
```

**Debugging:**
```cisco
R1# debug standby events
HSRP: Gi0/0 Grp 1 Hello in, Mismatched virtual IP address
HSRP:   Configured: 192.168.1.1, Received: 192.168.1.254
```

**Root Cause:** Virtual IP mismatch between R1 and R2

**Verification:**
```cisco
R1# show run interface gig0/0 | include standby
 standby 1 ip 192.168.1.1

R2# show run interface gig0/0 | include standby
 standby 1 ip 192.168.1.254  (WRONG!)
```

**Solution:**
```cisco
R2(config)# interface gig0/0
R2(config-if)# no standby 1 ip 192.168.1.254
R2(config-if)# standby 1 ip 192.168.1.1
```

**Lesson Learned:** Virtual IP, group number, and version must match exactly!

---

### Problem 2: Preemption not working

**Symptom:** R1 (priority 110) comes back online but R2 stays active

**Debugging:**
```cisco
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110   Standby  192.168.1.3     local           192.168.1.1
                    ^
              No 'P' = Preemption disabled!
```

**Root Cause:** Forgot to enable preemption on R1

**Solution:**
```cisco
R1(config)# interface gig0/0
R1(config-if)# standby 1 preempt
```

**Verify:**
```cisco
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           192.168.1.3     192.168.1.1
                    ^
              Preemption enabled, R1 now active!
```

**Lesson Learned:** Always enable preemption or higher-priority router won't reclaim active!

---

### Problem 3: PCs can't reach HSRP virtual IP

**Symptom:**
```
C:\> ping 192.168.1.1
Request timed out
```

**But HSRP shows active:**
```cisco
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           192.168.1.3     192.168.1.1
```

**Root Cause:** HSRP version mismatch or multicast blocked

**Verification:**
```cisco
R1# show run interface gig0/0 | include standby
 standby version 2

R2# show run interface gig0/0 | include standby
 (no standby version = defaults to version 1!)  ← Mismatch!
```

**HSRPv1 vs HSRPv2:**
- Version 1: Multicast 224.0.0.2, max 255 groups
- Version 2: Multicast 224.0.0.102, max 4095 groups

**Solution:**
```cisco
R2(config)# interface gig0/0
R2(config-if)# standby version 2
```

**Lesson Learned:** Always explicitly set HSRP version on both routers!

---

### Problem 4: Tracking causes constant flapping

**Symptom:** HSRP state changes every 30 seconds

**Debugging:**
```cisco
R1# show standby | include state changes
  23 state changes, last state change 00:00:15  ← Too many!
```

**Root Cause:** Tracked interface flapping (unstable WAN link)

**Verification:**
```cisco
R1# show track 1
Track 1
  Interface GigabitEthernet0/1 line-protocol
  Line protocol is Up
    47 changes, last change 00:00:08  ← Flapping!
```

**Solution:** Add delay to tracking
```cisco
R1(config)# track 1 interface gig0/1 line-protocol
R1(config-track)# delay down 10 up 30
R1(config-track)# exit
```

**What this does:**
- **Delay down 10:** Wait 10 seconds before considering link down
- **Delay up 30:** Wait 30 seconds before considering link up
- Prevents flapping from brief outages

**Lesson Learned:** Always add tracking delays for unstable links!

---

### Problem 5: Authentication failure

**Symptom:**
```cisco
R1# show standby
  Authentication MD5, key-string "HSRP$ecur3!"

R2# show standby
  Authentication MD5, key-string "hsrp$ecur3!"  ← Different case!
```

**Result:** HSRP neighbors don't form (authentication mismatch)

**Solution:** Passwords are case-sensitive!
```cisco
R2(config)# interface gig0/0
R2(config-if)# no standby 1 authentication
R2(config-if)# standby 1 authentication md5 key-string HSRP$ecur3!
```

**Lesson Learned:** HSRP authentication keys are case-sensitive!

---

## 📝 Key Takeaways

### HSRP Concepts Mastered

✅ **First Hop Redundancy Protocol (FHRP):**
- Eliminates single point of failure (gateway router)
- Virtual IP shared between routers
- Sub-second failover (with optimized timers)

✅ **HSRP Roles:**
- **Active:** Forwards traffic, responds to ARP for virtual IP
- **Standby:** Monitors active, ready to take over
- **Listening:** Other routers in group (if >2 routers)

✅ **HSRP States:**
1. **Initial:** Interface just enabled
2. **Learn:** Waiting to hear from active router
3. **Listen:** Knows virtual IP, waiting
4. **Speak:** Participating in election
5. **Standby:** Backup router
6. **Active:** Forwarding traffic

✅ **Priority and Preemption:**
- **Priority 0-255:** Higher = preferred active (default 100)
- **Preemption:** Higher-priority router reclaims active role
- **Tracking:** Decreases priority if monitored interface fails

✅ **HSRP Versions:**
- **Version 1:** 0-255 groups, multicast 224.0.0.2
- **Version 2:** 0-4095 groups, multicast 224.0.0.102, IPv6 support

### Technical Skills Showcased

| Skill | Implementation |
|-------|----------------|
| **High Availability Design** | Redundant gateway routers |
| **HSRP Configuration** | Virtual IP, priority, preemption |
| **Failover Optimization** | Hello/hold timers (1s/3s) |
| **Security** | MD5 authentication |
| **Intelligent Failover** | Interface tracking (WAN monitoring) |
| **Load Balancing** | Multiple HSRP groups |
| **Troubleshooting** | State analysis, debugging |

### Real-World Applications

**Use Cases:**
- **Enterprise Campus:** Data center gateway redundancy
- **Branch Offices:** Internet gateway redundancy
- **Service Providers:** Customer edge redundancy
- **Critical Infrastructure:** 99.99% uptime requirement

**Deployment Scenarios:**
- **Active/Standby:** One router forwards, other waits (this lab)
- **Active/Active:** Multiple HSRP groups for load distribution
- **Multi-Vendor:** VRRP (standard alternative to Cisco HSRP)

**Business Impact:**
- **Downtime Cost:** $5,600/minute for large enterprises
- **HSRP Failover:** 3 seconds = $280 loss
- **No Redundancy:** 30 minute outage = $168,000 loss
- **ROI:** HSRP pays for itself in first prevented outage!

---

## 🔗 Related Concepts

### What I Learned

**FHRP Protocols:**
- **HSRP (Cisco):** This lab, proprietary
- **VRRP (Standard):** RFC 3768, vendor-neutral
- **GLBP (Cisco):** Gateway Load Balancing, true load sharing

**Advanced HSRP Features:**
- **HSRP SSO (Stateful Switchover):** Preserves connections during failover
- **HSRP Object Tracking:** Track multiple conditions (interface, route, IP SLA)
- **HSRP BFD:** Bidirectional Forwarding Detection (sub-second failover)

**Integration with Other Technologies:**
- **HSRP + STP:** Align root bridge with HSRP active
- **HSRP + Routing Protocols:** Track OSPF/EIGRP neighbor status
- **HSRP + Firewall:** Stateful failover between ASA pairs

### Skills to Build Next

1. **VRRP Configuration** - Industry-standard alternative
2. **GLBP (Gateway Load Balancing)** - True active/active
3. **HSRP SSO** - Stateful failover
4. **IP SLA Tracking** - Monitor end-to-end reachability
5. **Firewall High Availability** - ASA active/standby

### Career Relevance

**Roles This Prepares You For:**
- Network Engineer (HSRP is day-1 requirement)
- Data Center Engineer
- Network Architect
- Site Reliability Engineer (SRE)
- Infrastructure Engineer

**Interview Questions Answered:**
- "How do you provide gateway redundancy?"
- "Explain HSRP failover process."
- "What's the difference between HSRP and VRRP?"
- "How would you achieve sub-second failover?"
- "Troubleshoot: HSRP active/active (split-brain) - what happened?"

---

## 🔗 Related Labs

- [Lab 4: OSPF Multi-Area Design](../routing/lab04-ospf-multi-area.md) - Previous
- [Lab 6: Network Troubleshooting Scenario](../troubleshooting/lab06-broken-network.md) - Next
- [Lab 7: QoS Implementation for VoIP](../qos/lab07-voip-qos.md) 

---

## 📚 Resources & References

**Cisco Documentation:**
- [HSRP Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipapp_fhrp/configuration/xe-16/fhp-xe-16-book/fhp-hsrp.html)
- [HSRP Object Tracking](https://www.cisco.com/c/en/us/support/docs/ip/hot-standby-router-protocol-hsrp/13780-6.html)

**RFCs:**
- RFC 2281 - Cisco HSRP (informational)
- RFC 3768 - VRRP (standard alternative)

**Comparison:**
- HSRP vs VRRP vs GLBP feature matrix
- When to use each FHRP protocol

---

**Lab Completed:** February 2026  
**Source:** Custom lab based on CCNP ENCOR + production deployments  
**Time Spent:** 3 hours (including failover testing and optimization)  
**Difficulty:** Intermediate  
**Prerequisites:** Basic routing, interface configuration, IP addressing

---

**💡 Real-World Success Metric:**

In interviews, quantify the impact:
- "Implemented HSRP for 500-user network, achieving 99.98% gateway uptime"
- "Reduced failover time from 30 seconds to 3 seconds with timer optimization"
- "Prevented $168K in downtime costs with redundant gateway design"

This separates you from candidates who just know the commands - you understand **business value**!
