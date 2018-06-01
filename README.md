# H1 Installing Openstack Pike/Queens on Oracle Virtualbox


On Ubuntu 18.04 host, install virtualbox 5.2.x
Open Virtualbox. Click File --> Host Network Manager --> Create . You should see vboxnet0. Check Enable DHCP Server
Enable NAT and forwarding for the newly created network and interface



echo 1 > /proc/sys/net/ipv4/ip_forward

iptables --table nat --append POSTROUTING --out-interface enp2s0 -j MASQUERADE

iptables --append FORWARD --in-interface vboxnet0 -j ACCEPT



Create a virtualbox vm with 6 cpu cores, 8 GB memory and 100GB disk
Under Network Settings of the vm, change from NAT to Host-only Adapter. It should select vboxne0 by default. Click Advanced and Select Allow All in Promiscuous Mode
Power on the vm, run 'yum update -y" and reboot
Edit /etc/sysconfig/network-scripts/ifcfg-enp0s3 and change DHCP to manual IP . The entry should look like



BOOTPROTO='none'

IPADDR='192.168.56.12'

NETMASK='255.255.255.0'

GATEWAY='192.168.56.1'

DNS1='10.58.8.50'



systemctl restart network
yum install wget net-tools telnet vim -y
yum install -y centos-release-openstack-queens
yum update -y
yum install -y openstack-packstack
systemctl stop firewalld
systemctl disable firewalld
systemctl stop NetworkManager
systemctl disable NetworkManager
setenforce 0 / Edit /etc/selinux/config and disable it permanently 
packstack --gen-answer-file=/root/answer.txt
Edit the following sections in answer.txt



CONFIG_PROVISION_DEMO=n

CONFIG_KEYSTONE_ADMIN_PW=openstack



Run the installer: packstack --answer-file /root/answer.txt
After successful install under /etc/sysconfig/network-scripts create the following two files


ifcfg-br-ex

DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
DNS1=10.58.8.50
ONBOOT=yes
IPADDR=192.168.56.12
PREFIX=24
GATEWAY=192.168.56.1

ifcfg-enp0s3

DEVICE=enp0s3
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes

systemctl restart network
Enable Promiscuous mode on host interface : ip link set enp0s3 promisc on
source keystone_admin
Cluster status: nova service-list
Create networks and a router



neutron net-create public --provider:network_type flat --provider:physical_network extnet --router:external

neutron subnet-create public --name public_subnet --allocation-pool start=192.168.56.100,end=192.168.56.200 --disable-dhcp --gateway 192.168.56.1 192.168.56.0/24

neutron router-create router1 --ha False

neutron router-gateway-set router1 public

neutron router-interface-add router1 private_subnet



Check router port status



neutron router-list --fields name --fields id

neutron router-port-list id

neutron port-show id



If you need to restart the router



neutron port-update routerid --admin-state-up False

neutron port-update routerid --admin-state-up True



Login to dashboard and create virtual machines
If you see No valid host found error, run: systemctl restart openstack-nova-compute.service and create the instance again
Troubleshooting neutron



grep -E -i "error|trace" /var/log/neutron/server.log

cat /etc/neutron/plugins/ml2/openvswitch_agent.ini and make sure bridge mapping looks exactly like 'bridge_mappings=extnet:br-ex
