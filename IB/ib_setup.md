There is one IB card with two ports on deepstorage, and four IB cards, each of which has one port on dgx. 

[Set up drivers](#setupdriver) [Config network interface](#config_network) need to be done for both servers. 
## SetUp drivers<a name="setupdriver"></a>
- Verify that cards are installed correctly and are recognized by the system
```lspci | grep -i mellanox
```

- Verify that the infiniband drivers are present (Mellanox Adapters' Linux VPI Drivers for Ethernet and InfiniBand 
are also available Inbox in all the major distributions, RHEL, SLES, Ubuntu and more)
```lsmod | grep -i ib_
```

- Verify that the Mellanox Software Tools (MST) / MLNX_OFED drivers were installed correctly
Need to make sure the version is correct. Use the driver in /home/infiniband/ on dgx or the driver in /tmp/ on deepstorage.  
Run the ``install.sh`` to install MST, and this will remove any existing versions.  

- Check the version of MST 
```
modinfo mlx5_core | grep -i version | head -1
```

- Load MST software
```
sudo mst start
```

- Restart the IB service (may need to restart here)
```
sudo service openibd restart
```

- Make sure the link type is Ethernet
In this guide, page 50 describes how to change link type [here](https://images.nvidia.com/content/technologies/deep-learning/pdf/DGX-1-UserGuide.pdf)
```
ibv_devinfo | grep -e "hca_id\|link_layer"
```

- Add following to ``/etc/modules`` for both server and client. Need to reboot
```
mlx4_ib
ib_umad
ib_ipoib
```

- Open subnet manager
```
/etc/init.d/opensmd start
/etc/init.d/opensmd status
```

The infiniband ports should be active and linkup.

## Config network interface <a name="config_network"></a>
- Since we are using ethernet over infiniband, we need to assign subnet IP address to IB ports.
```
# Show all network interfaces, even if they are down. 
ifconfig -a
```

- Figure out which interface (from the ifconfig command) corresponds to which port (from the ibstat command), since we
need to figure out which ports are active and linkup
```
/sys/class/infiniband/mlx5_0/device/net/enp5s0
```
For now (Feb 19, 2020), the mlx5_0 is enp5s0 and mlx5_1 is enp12s0

- Edit ``/etc/netplan/<*.yaml>`` file. Apply netplan changes.
```
sudo netplan apply
```

Netplan file on dgx
```
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  #renderer: networkd
  bonds:
    bond1:
      addresses: [ 192.168.1.2/24 ]
      dhcp4: false
      interfaces: [ enp5s0, enp12s0 ]
      mtu: 9000  
      parameters:
        mode: active-backup
        primary: enp5s0
      routes:
      - to: 192.168.1.0/24
        via: 192.168.1.2
  ethernets:
    enp1s0f0:
      addresses: [ 10.1.122.12/26 ]
      gateway4: 10.1.122.1
      nameservers:
        search: [ ad.medctr.ucla.edu ]
        addresses:
          - 10.4.14.10
          - 10.6.14.10
    enp132s0:
      dhcp4: false
      dhcp6: false
    enp139s0:
      dhcp4: false
      dhcp6: false
    enp12s0:
      dhcp4: false
      dhcp6: false
    enp5s0:
      dhcp4: false
      dhcp6: false
```

Netplan on deepstorage
```
# This file is generated from information provided by
# the datasource.  Changes to it will not persist across an instance.
# To disable cloud-init's network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    bonds:
        bond1:
            addresses:
            - 192.168.1.6/24
            dhcp4: false
            interfaces:
            - enp175s0f0
            - enp175s0f1
            mtu: 9000
            parameters:
                mode: active-backup
                primary: enp175s0f0
            routes:
            - to: 192.168.1.0/24
              via: 192.168.1.6
    ethernets:
        eno1:
            addresses:
            - 10.1.122.26/24
            dhcp4: false
            gateway4: 10.1.122.1
            nameservers:
                addresses:
                - 10.4.14.10
                - 10.6.14.10
                search:
                - mednet.ucla.edu
        eno2:
            addresses:
            - 192.168.0.2/24   
            dhcp4: false
            routes:
            - to: 192.168.0.0/24
              via: 192.168.0.2
        enp175s0f0:
            dhcp4: false
            dhcp6: false
        enp175s0f1:
            dhcp4: false
            dhcp6: false
    version: 2
```
## Config NFS over Infiniband




mtu: 65520