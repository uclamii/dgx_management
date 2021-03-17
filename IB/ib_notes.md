There is one IB card with two ports on deepstorage, and four IB cards, each of which has one port on dgx. 

[Set up drivers](#setupdriver) [Config network interface](#config_network) need to be done for both servers. 
For [NFS over IB](#nfs_over_ib), different steps are required for server (deepstorage) and client (dgx).  

## Common steps to solve IB issues
- Check the status of IB connection

```
ibstat
```
- Make sure the "rdma 20049" port is added in `/proc/fs/nfsd/portlist`. The "rdma 20049" might be removed after deepstorage reboot, after NFS server restart, ...
This may stop the IB from working, and is the most commonly encountered issue for me.   
```
# Perform this on deepstorage!
echo "rdma 20049" | sudo tee /proc/fs/nfsd/portlist
```

- Test data transform
```
dd of=/raid/jiayunli/data/tempt if=/raid/jiayunli/data/storage_slides/finished_011619/11223.svs bs=1000M count=1024 oflag=direct
listCard 

- More information may be found [here](https://docs.google.com/document/d/14BfhDKKwjJkztUdh5guTjcPpxoFoRZfK4Yw1Rk9bM7w/edit?usp=sharing)

## SetUp drivers<a name="setupdriver"></a>.  
- Verify that cards are installed correctly and are recognized by the system
```
lspci | grep -i mellanox
```
- Verify that the infiniband drivers are present (Mellanox Adapters' Linux VPI Drivers for Ethernet and InfiniBand 
are also available Inbox in all the major distributions, RHEL, SLES, Ubuntu and more)
```
lsmod | grep -i ib_
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
## Config NFS over Infiniband <a name="nfs_over_ib"></a>
- Check if the NFS server is installed (deepstorage)
```dpkg -l | grep nfs-kernel-server```

- Install the required packages (deepstorage)
```apt-get install nfs-kernel-server```

- Install NFS client (dgx)
```sudo apt-get install rpcbind nfs-common```

- Edit the ``/etc/exports`` file to export folder to be mounted on the host server (deepstorage)

- Load RDMA transport module (deepstorage)
``` modprobe svcrdma```

- Make sure the server is listening on the RDMA transport port (deepstorage)
``` echo rdma 20049 > /proc/fs/nfsd/portlist```

- Load the RDMA transport module on the client (dgx)
```modprobe xprtrdma```

- Mount the folder (dgx)
The folder on deepstorage should be exported as specified in the ``/etc/exports``
```
sudo mount -o rdma,port=20049 192.168.1.6:<folder on deepstorage> <folder on dgx>
```

- Check the mount type (dgx)
```
mount|grep storage_slides
192.168.1.6:/data/jiayun/slides_deid on /raid/jiayunli/data/storage_slides type nfs4 (rw,relatime,vers=4.2,rsize=1048576,wsize=1048576,namlen=255,hard,proto=rdma,port=20049,timeo=600,retrans=2,sec=sys,clientaddr=192.168.1.2,local_lock=none,addr=192.168.1.6)
```

- Mount the folder automatically during boot (dgx)
Need to edit the ``/etc/fstab`` to add mount folder options.

```
# Mount deepstorage
192.168.1.6:/data/jiayun/slides_deid /raid/jiayunli/data/storage_slides nfs rw,noatime,proto=rdma,port=20049,rsize=1048576,wsize=1048576,nolock,intr,fsc,nofail 0 0
```

- Test the file transfer speed
```dd of=<folder on dgx> if=<file from mounted folder on dgx> bs=1000M count=1024 oflag=direct```

- Test the IB link only (no disk operation)
1) Start the ib server on one server (deepstorage) ``ib_send_bw``
2) Test speed on another server (dgx) ``ib_send_bw -d mlx5_0 -i 1 -F --report_gbits 192.168.1.6``

## References and useful tutorials
- [Test IB speed](https://community.mellanox.com/s/article/ib-send-bw)
- [Config IB link type](https://docs.nvidia.com/dgx/dgx1-user-guide/configuring-managing-dgx1.html#infiniband-port-changing)
- [Config IPoIB](https://furneaux.ca/wiki/IPoIB)
- [Config static ip](https://websiteforstudents.com/configure-static-ip-addresses-on-ubuntu-18-04-beta/)
- [Config NFS on DGX](https://docs.nvidia.com/dgx/dgx1-user-guide/preparing-for-using-containers.html#setting-up-nfs)
- [Config RDMA over NFS](https://community.mellanox.com/s/article/howto-configure-nfs-over-rdma--roce-x)x
