# Installing Openstack Queens on Oracle Virtualbox


* On Ubuntu 18.04 host, install virtualbox 5.2.x
* Open Virtualbox. Click File --> Host Network Manager --> Create . You should see vboxnet0. Check Enable DHCP Server
* Enable NAT and forwarding for the newly created network and interface
    
    ```sh
    echo 1 > /proc/sys/net/ipv4/ip_forward
    iptables --table nat --append POSTROUTING --out-interface enp2s0 -j MASQUERADE
    iptables --append FORWARD --in-interface vboxnet0 -j ACCEPT
    ```

* Create a CentOS 7 64 bit virtualbox vm with 6 cpu cores, 8 GB memory and 100GB disk
* Under Network Settings of the vm, change from NAT to Host-only Adapter. It should select vboxnet:0 by default. Click Advanced and Select Allow All in Promiscuous Mode
* Power on the vm, run 'yum update -y" and reboot
* Edit /etc/sysconfig/network-scripts/ifcfg-enp0s3 and change DHCP to manual IP . The entry should look like

    ```sh
    BOOTPROTO='none'
    IPADDR='192.168.56.12'
    NETMASK='255.255.255.0'
    GATEWAY='192.168.56.1'
    DNS1='8.8.8.8'
    ```
    
* systemctl restart network
* yum install wget net-tools telnet vim -y
* yum install -y centos-release-openstack-queens
* yum update -y
* yum install -y openstack-packstack
* systemctl stop firewalld
* systemctl disable firewalld
* systemctl stop NetworkManager
* systemctl disable NetworkManager
* setenforce 0 / Edit /etc/selinux/config and disable it permanently 
* packstack --gen-answer-file=/root/answer.txt
* Edit the following sections in answer.txt
    ```sh
    CONFIG_PROVISION_DEMO=n
    CONFIG_KEYSTONE_ADMIN_PW=openstack
    ```
* Run the installer: packstack --answer-file /root/answer.txt
* After successful install under /etc/sysconfig/network-scripts create the following two files

    ifcfg-br-ex:
    ```sh
    DEVICE=br-ex
    DEVICETYPE=ovs
    TYPE=OVSBridge
    BOOTPROTO=static
    DNS1=8.8.8.8
    ONBOOT=yes
    IPADDR=192.168.56.12
    PREFIX=24
    GATEWAY=192.168.56.1
    ```
    
    ifcfg-enp0s3:
    
    ```sh
    DEVICE=enp0s3
    TYPE=OVSPort
    DEVICETYPE=ovs
    OVS_BRIDGE=br-ex
    ONBOOT=yes
    ```

* systemctl restart network
* Enable Promiscuous mode on host interface : ip link set enp0s3 promisc on
* source keystone_admin
* Cluster status: nova service-list
* Create networks and a router

    ```sh
    # Create a public/external network
    neutron net-create public --provider:network_type flat --provider:physical_network extnet --router:external
    
    neutron subnet-create public --name public_subnet --allocation-pool start=192.168.56.100,end=192.168.56.200 --disable-dhcp --gateway 192.168.56.1 192.168.56.0/24
    
    # Create private network
    neutron net-create private
    
    neutron subnet-create private --name private_subnet --allocation-pool start=10.10.1.100,end=10.10.1.200 10.10.1.0/24

    # Create router
    neutron router-create router1 --ha False

    # Set interfaces
    neutron router-gateway-set router1 public

    neutron router-interface-add router1 private_subnet
    ```

* Check router port status

    ```sh
    neutron router-list --fields name --fields id

    neutron router-port-list id

    neutron port-show id
    ```

* If you need to restart the router
    
    ```sh
    neutron port-update routerid --admin-state-up False
    neutron port-update routerid --admin-state-up True
    ```
* Login to dashboard at http://192.168.56.12/dashboard using admin/openstack nd create virtual machines

* Cirros cloud image download link http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img . Username: cirros Password: gocubsgo

* After you login to the dashboard, enable ICMP and other required tcp ports

* If you see No valid host found error, run: systemctl restart openstack-nova-compute.service and create the instance again

* Troubleshooting neutron
    ```sh
    grep -E -i "error|trace" /var/log/neutron/server.log
    cat /etc/neutron/plugins/ml2/openvswitch_agent.ini and make sure bridge mapping     looks exactly like 'bridge_mappings=extnet:br-ex
    ```

* If your virtualbox runs on MacOS add the following in /etc/pf.conf and run pfctl -e -f /etc/pf.conf. In this example en3 and en0 are my ethernet and wifi interfaces
    ```sh
    scrub-anchor "com.apple/*"
    nat-anchor "com.apple/*"
    rdr-anchor "com.apple/*"
    nat on {en3, en0} proto {tcp, udp, icmp} from 192.168.56.0/24 to any -> {en3, en0}
    pass from {lo0, 192.168.56.0/24} to any keep state
    dummynet-anchor "com.apple/*"
    anchor "com.apple/*"
    load anchor "com.apple" from "/etc/pf.anchors/com.apple"
    ```
* NAT on Windows 10
    ```sh
    New-NetNat –Name NATNetwork –InternalIPInterfaceAddressPrefix 192.168.56.0/24 –Verbose
    ```
