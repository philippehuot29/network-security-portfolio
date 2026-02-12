# Lab 41: VLAN Configuration and Inter-VLAN Routing

## 🎯 Objective
Configure VLANs on a Cisco switch to segment network traffic and enable communication between VLANs using router-on-a-stick (inter-VLAN routing).

## 🔧 Equipment Used
- 1 Cisco Router (Router0)
- 1 Cisco Switch (Switch0)
- 3 PCs (PC0, PC1, PC2)
- Software: Cisco Packet Tracer

## 📋 Topology
```
PC0 (VLAN 10 - Sales)     ---|
PC1 (VLAN 20 - Engineering)---|--- Switch0 --- Router0 (Router-on-a-Stick)
PC2 (VLAN 10 - Sales)     ---|

Router0 Subinterfaces:
- Gig0/0.10: 192.168.10.1 (VLAN 10 gateway)
- Gig0/0.20: 192.168.20.1 (VLAN 20 gateway)
```

## ⚙️ Configuration Steps

### Step 1: Create VLANs on Switch
```cisco
Switch> enable
Switch# configure terminal
Switch(config)# vlan 10
Switch(config-vlan)# name Sales
Switch(config-vlan)# exit

Switch(config)# vlan 20
Switch(config-vlan)# name Engineering
Switch(config-vlan)# exit
```

### Step 2: Assign Switch Ports to VLANs
```cisco
! Assign PC0 to VLAN 10 (Sales)
Switch(config)# interface fa0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# exit

! Assign PC1 to VLAN 20 (Engineering)
Switch(config)# interface fa0/2
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 20
Switch(config-if)# exit

! Assign PC2 to VLAN 10 (Sales)
Switch(config)# interface fa0/3
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# exit
```

### Step 3: Configure Trunk Port to Router
```cisco
! Configure port connecting to router as trunk
Switch(config)# interface gig0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# exit
```

### Step 4: Verify VLAN Configuration
```cisco
Switch# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Fa0/4, Fa0/5, ... Gig0/2
10   Sales                            active    Fa0/1, Fa0/3
20   Engineering                      active    Fa0/2
```

### Step 5: Configure Router Subinterfaces (Router-on-a-Stick)
```cisco
Router> enable
Router# configure terminal

! Configure subinterface for VLAN 10
Router(config)# interface gig0/0.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 192.168.10.1 255.255.255.0
Router(config-subif)# exit

! Configure subinterface for VLAN 20
Router(config)# interface gig0/0.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 192.168.20.1 255.255.255.0
Router(config-subif)# exit

! Enable physical interface
Router(config)# interface gig0/0
Router(config-if)# no shutdown
Router(config-if)# exit
```

### Step 6: Configure PC IP Addresses

**PC0 (VLAN 10 - Sales):**
- IP Address: 192.168.10.10
- Subnet Mask: 255.255.255.0
- Default Gateway: 192.168.10.1

**PC1 (VLAN 20 - Engineering):**
- IP Address: 192.168.20.10
- Subnet Mask: 255.255.255.0
- Default Gateway: 192.168.20.1

**PC2 (VLAN 10 - Sales):**
- IP Address: 192.168.10.11
- Subnet Mask: 255.255.255.0
- Default Gateway: 192.168.10.1

## ✅ Testing

### Test 1: Same VLAN Communication (VLAN 10)
```
PC0> ping 192.168.10.11
Result: ✅ SUCCESS (PC0 can reach PC2 - both in VLAN 10)
```

### Test 2: Inter-VLAN Communication (VLAN 10 → VLAN 20)
```
PC0> ping 192.168.20.10
Result: ✅ SUCCESS (PC0 can reach PC1 via router)
```

### Test 3: Different VLAN Without Routing (Before Router Config)
```
PC0> ping 192.168.20.10
Result: ❌ FAILED (Expected - VLANs are isolated without routing)
```

## 🐛 Possible Troubleshooting Issues

### Problem 1: PCs in same VLAN couldn't communicate
**Symptom:** PC0 couldn't ping PC2 even though both were in VLAN 10

**Root Cause:** I forgot to set the switch port mode to `access` - ports were still in default mode

**Solution:** 
```cisco
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
```

**Lesson Learned:** Always explicitly set port mode. Use `show interfaces switchport` to verify configuration.

---

### Problem 2: Inter-VLAN routing not working
**Symptom:** PC0 couldn't ping PC1 (different VLANs)

**Root Cause:** Forgot to enable the physical router interface with `no shutdown`

**Solution:**
```cisco
Router(config)# interface gig0/0
Router(config-if)# no shutdown
```

**Lesson Learned:** After configuring subinterfaces, always remember to enable the physical interface. Subinterfaces inherit the shutdown state.

---

### Problem 3: Trunk not passing VLAN traffic
**Symptom:** Traffic worked on access ports but not across VLANs

**Root Cause:** Didn't configure trunk port on switch side

**Solution:**
```cisco
Switch(config)# interface gig0/1
Switch(config-if)# switchport mode trunk
```

**Verification:**
```cisco
Switch# show interfaces trunk
```

**Lesson Learned:** Both ends of a trunk link need to be configured as trunk. Always verify with `show interfaces trunk`.

## 📝 Key Takeaways

✅ **VLANs provide Layer 2 network segmentation** - devices in different VLANs can't communicate without a Layer 3 device (router)

✅ **Access ports** belong to ONE VLAN, **Trunk ports** carry traffic for MULTIPLE VLANs (tagged with 802.1Q)

✅ **Router-on-a-stick** uses subinterfaces to route between VLANs - one physical interface, multiple logical interfaces

✅ **Encapsulation dot1Q [vlan-id]** is required on router subinterfaces to match switch VLAN tagging

✅ **Always verify configurations:**
- `show vlan brief` - check VLAN assignments
- `show interfaces switchport` - verify port mode (access/trunk)
- `show interfaces trunk` - verify trunk operation
- `show ip interface brief` - verify router interface status

✅ **Security consideration:** Default VLAN 1 should not be used for production traffic - create separate VLANs for each department/function

## 🔗 Related Concepts

**What I Learned:**
- **802.1Q VLAN tagging:** How switches add VLAN tags to frames
- **Native VLAN:** Untagged VLAN on trunk (VLAN 1 by default)
- **VLAN hopping attacks:** Why you should change native VLAN
- **DTP (Dynamic Trunking Protocol):** Why manual trunk config is safer

**Next Steps:**
- Practice with more complex topologies (multiple switches)
- Learn about VTP (VLAN Trunking Protocol)
- Explore Layer 3 switches (faster inter-VLAN routing)
- Study VLAN security best practices

## 🔗 Related Labs

---

**Lab Completed:** January 2026  
**Source:** 101 Labs - CompTIA Network+ 
**Time Spent:** 1 hours (including troubleshooting)
```

---
