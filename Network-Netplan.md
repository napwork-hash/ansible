# Troubleshoot Network Config

## #1 Problem stuck installation on ubuntu 22.04 at kernel

```
1. When creating the VM, don't add any network device
2. Install ubuntu server
3. Shut down the VM
4. Edit the hardware to add a network device
5. Start the VM and edit or create the network config file to add your NIC
6. Test the network connection to ensure it's working
```


## #2 Problem no internet setelah pemasangan Network

Cek:

```
ls /etc/netplan/
```

Biasanya ada ```50-cloud-init.yaml```

isinya cuma ada:

```
network:
  version: 2
```

buat file baru dengan nama ```01-static.yaml``` di ```/etc/netplan/01-static.yaml```

config:

```
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      optional: true
```

atau kalau mau static

```
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.1.101/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Nonaktifkan config otomatis (buat disable file 50-cloud-init.yaml)

```
sudo vim /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

```
network: {config: disabled}
```

atau

```
sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg << 'EOF'
network: {config: disabled}
EOF
```

Terakhir baru  ```sudo netplan try``` dan ```sudo netplan apply```

Test restart VM ```sudo reboot now```