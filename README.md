
# Part 1: Configure the ASA 5506-X
## Step 1: Configure Basic Settings on the ASA device

# HQ-ASA5506
enable
Thecar1Admin
conf term
domain-name thecar1.com
hostname HQ-ASA5506
interface g1/1
nameif OUTSIDE
security-level 1
ip address 209.165.200.253 255.255.255.240
no shutdown
interface g1/2
nameif INSIDE
security-level 100
ip address 192.168.10.1 255.255.255.0
no shutdown
interface g1/3
nameif DMZ
security-level 70
ip address 192.168.20.1 255.255.255.0
no shutdown
exit


# Step 2: Configure the DHCP service on the ASA device for the internal network.
# HQ-ASA5506
dhcpd address 192.168.10.25-192.168.10.35 INSIDE
dhcpd dns 192.168.10.10 interface INSIDE
dhcpd option 3 ip 192.168.10.1
dhcpd enable INSIDE

# Step 3: Configure routing on the ASA.
# HQ-ASA5506
route OUTSIDE 0.0.0.0 0.0.0.0 209.165.200.254

# Step 4: Configure Secure Network Management for the ASA Device.
# a. Configure the ASA with NTP and AAA:
# HQ-ASA5506
ntp authenticate
ntp authentication-key 1 md5 corpkey
ntp server 192.168.10.10
ntp trusted-key 1

# b. Configure AAA and SSH
username Car1Admin password adminpass01
aaa authentication ssh console LOCAL
crypto key generate rsa modulus 1024
yes
ssh 192.168.10.250 255.255.255.255 INSIDE
ssh timeout 20

# Step 5: Configure NAT Service for the ASA device for both INSIDE and DMZ networks.
# HQ-ASA5506
object network INSIDE-nat
subnet 192.168.10.0 255.255.255.0
nat (inside,outside) dynamic interface
exit
configure terminal
object network DMZ-web-server
host 192.168.20.2
nat (dmz,outside) static 209.165.200.241
exit
configure terminal
object network DMZ-dns-server
host 192.168.20.5
nat (dmz,outside) static 209.165.200.242
exit

# Step 6: Configure ACL on the ASA device to implement the Security Policy.
# HQ-ASA5506
configure terminal
access-list NAT-IP-ALL extended permit ip any any
access-group NAT-IP-ALL in interface OUTSIDE
access-group NAT-IP-ALL in interface DMZ
access-list OUTSIDE-TO-DMZ extended permit tcp any host 209.165.200.241 eq 80
access-list OUTSIDE-TO-DMZ extended permit tcp any host 209.165.200.242 eq 53
access-list OUTSIDE-TO-DMZ extended permit udp any host 209.165.200.242 eq 53
access-list OUTSIDE-TO-DMZ extended permit tcp host 198.133.219.35 host 209.165.200.241 eq
ftp
end
copy running-config startup-config

## Part 2: Configure Layer 2 Security on a Switch
# Step 1: Disable Unused Switch Ports
# Switch 1
enable
conf t
interface range f0/2-4, f0/6-9, f0/11-22, g0/2
shutdown
switchport mode access
switchport nonegotiate


# Step 2: Implement Port Security
# Switch 1
interface range f0/1, f0/5, f0/10
switchport mode access
switchport port-security
switchport port-security maximum 2
switchport port-security mac-address sticky
switchport port-security violation restrict
switchport nonegotiate

# Step 3: Implement STP Security
# Switch 1
interface range f0/1, f0/5, f0/10, g0/1
spanning-tree bpduguard enable
spanning-tree portfast
end
copy running-config startup-config

# Part 3: Configure a Site-to-Site IPsec VPN between the HQ and the Branch Routers
# HQ Router
enable
Password: RTR-AdminP@55
conf ter
access-list 120 permit ip 209.165.200.240 0.0.0.15 198.133.219.32 0.0.0.31
crypto isakmp policy 10
encryption aes 256
hash sha
authentication pre-share
group 2
lifetime 1800
exit
crypto isakmp key Vpnpass101 address 198.133.219.2
crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac
crypto map VPN-MAP 10 ipsec-isakmp
match address 120
set transform-set VPN-SET
set peer 198.133.219.2
set pfs group2
set security-association lifetime seconds 1800
exit
int s0/0/0
crypto map VPN-MAP
end
copy running-config startup-confi

# Branch Router
enable
Password: RTR-AdminP@55
conf ter
access-list 120 permit ip 198.133.219.32 0.0.0.31 209.165.200.240 0.0.0.15
crypto isakmp policy 10
encryption aes 256
hash sha
authentication pre-share
group 2
lifetime 1800
exit
crypto isakmp key Vpnpass101 address 209.165.200.226
crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac
crypto map VPN-MAP 10 ipsec-isakmp
match address 120
set transform-set VPN-SET
set peer 209.165.200.226
set pfs group2
set security-association lifetime seconds 1800
exit
int s0/0/0
crypto map VPN-MAP
end
copy running-config startup-config


