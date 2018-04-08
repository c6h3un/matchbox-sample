# CoreOS install
Install container linux with matchbox.

## Installation Steps
- Download images
	- coreos
	- matchbox
	- dnsmasq
- Setup with configurations
- Run matchbox and dnsmasq
- Select `pxe network boot` option and install to disks


### Download the images
#### CoreOS
- check which version do you want to use on [release note](https://coreos.com/releases/)
- Run the script under assets
	```
	$ cd assets
	$ ./get-coreos stable 1576.5.0 .
	# you will see following structure when finished.
	$ tree -h
	.
	├── [4.0K]  coreos
	│   ├── [4.0K]  1353.8.0
	│   │   ├── ...
	│   │   └── ...
	│   └── [4.0K]  1576.5.0
	│       ├── [ 15K]  CoreOS_Image_Signing_Key.asc
	│       ├── [329M]  coreos_production_image.bin.bz2
	│       ├── [ 566]  coreos_production_image.bin.bz2.sig
	│       ├── [288M]  coreos_production_pxe_image.cpio.gz
	│       ├── [ 566]  coreos_production_pxe_image.cpio.gz.sig
	│       ├── [ 39M]  coreos_production_pxe.vmlinuz
	│       └── [ 566]  coreos_production_pxe.vmlinuz.sig
	└── [2.6K]  get-coreos
	```

#### Matchbox and Dnsmasq
- Pull the docker images
  ```
  $ docker pull quay.io/coreos/matchbox:latest
  $ docker pull quay.io/coreos/dnsmasq:latest
  ```
- (Offline install) may need to download the image
  ```
  $ mkdir docker-images
  $ docker save -o docker-images/matchbox_latest quay.io/coreos/matchbox:latest
  $ docker save -o docker-images/dnsmasq_latest quay.io/coreos/dnsmasq:latest

  $ docker load -i docker-images/dnsmasq_latest
  $ docker load -i docker-images/dnsmasq_latest
  ```
### Setup with configurations
The configs used here will not work, change the following informations according to your environment configs.
- Hostname
- Network(IP, gateway, interface)
- DNS nameserver, search domain
- NTP server
- Docker registry (secure / insecure)
- password
- ssh key

### Run matchbox static file server and dnsmasq dhcp server
#### Matchbox
- command
  ```
  $ docker run --rm \
    -p 8080:8080 \
    -v $PWD:/var/lib/matchbox:Z \
    quay.io/coreos/matchbox:latest \
      -address=0.0.0.0:8080 \
      -log-level=debug
  ```
- check
  ```
  $ curl http://localhost:8080/ignition?mac=00-00-00-00-00-00
  $ curl 'http://10.128.113.161:8080/ignition?uuid=b135a44c-53c0-eac4-f5b6-d414ee7b2cd8&=&os=installed'
  ```
  will get the right json file and 'level=debug msg="Matched an Ignition or Container Linux Config template" group=k8s-node-1 labels=map[: os:installed uuid:b135a44c-53c0-eac4-f5b6-d414ee7b2cd8] profile=k8s-node' in log

#### Dnsmasq
- environment file
  ```
  #!/bin/sh
  INTERFACE_NAME="eth0"
  DHCP_START="10.128.113.101"
  DHCP_END="10.128.113.150"
  MACHBOX_DOMAIN="matchbox.localnet"
  MACHBOX_IP="10.128.113.161"
  ```
- command
  ```
  $ source ./dnsmasq_env
  $ docker run --rm \
    --cap-add=NET_ADMIN \
    --net=host \
    --name=dnsmasq \
    quay.io/coreos/dnsmasq \
    -d -q -i $INTERFACE_NAME \
    --dhcp-range=$DHCP_START,$DHCP_END \
    --enable-tftp \
    --tftp-root=/var/lib/tftpboot \
    --dhcp-userclass=set:ipxe,iPXE \
    --dhcp-boot=tag:#ipxe,undionly.kpxe \
    --dhcp-boot=tag:ipxe,http://$MACHBOX_DOMAIN:8080/boot.ipxe \
    --address=/$MACHBOX_DOMAIN/$MACHBOX_IP \
    --log-queries \
    --log-dhcp
  ```
- check
  ```
  dig +short $MACHBOX_DOMAIN @$MACHBOX_IP
  ```
- stop
   ```
   docker stop dnsmasq
   ```

### Install to disks
 - Select `pxe network boot`
 - select disk type `ide` (VMs)

## References
- Container Linux Documentation
  - [Spec](https://coreos.com/os/docs/latest/configuration.html)
  - [Create user password](https://coreos.com/os/docs/latest/cloud-config.html#users)
  - [Configure Date and TimeZone](https://coreos.com/os/docs/latest/configuring-date-and-timezone.html)
- [Matchbox Documentation](https://coreos.com/matchbox/docs/latest/)
  - [Matchbox Concepts](https://coreos.com/matchbox/docs/latest/matchbox.html)
  - [Machine Life Cycle](https://coreos.com/matchbox/docs/latest/machine-lifecycle.html)
