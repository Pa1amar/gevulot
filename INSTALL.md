# Gevulot node installation guide

## Hardware system requirements

Gevulot node itself runs on modest resources. The real system HW requirements are dictated by the provers run on this platform.

Many ZK provers require modern multi-core processor and a lot of RAM. Some also need plenty of fast disk space.

Currently Gevulot supports nVidia 3090 and 4090 GPUs.

### Minimum requirements for devnet

- CPU: 64+ cores
- RAM: 512GB
- Disk: 1 TB NVMe

These requirements are formed by best estimate based on earlier documentation of production requirements for Polygon Hermez and Aztec provers, and empirical experience from working with Taiko prover. 

## Software system requirements

Gevulot node requires modern Linux distribution with systemd, quadlet and necessary KVM components (on Fedora this is provider by `qemu-kvm` package) installed.

Development & testing has been done on Fedora 39.

For GPU support the PCI passthrough must be appropriately configured. 

## Disk partitioning

Disk partitioning should be done in such a way that Gevulot data directory has most of the disk space. It is used for unikernel images, as well as for transaction input data.

## PCI passthrough for GPU

### Blacklist nVidia drivers from loading during boot

When passing GPU to VM running unikernel, the device cannot be in use by the host operating system. Therefore the GPU drivers must be blacklisted.

**/etc/modprobe.d/nvidia.conf**
```
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
blacklist nvidia-drm
```

On Ubuntu the module blacklisting doesn't necessarily work as expected. In that case [driverctl](https://gitlab.com/driverctl/driverctl) can help:
```
sudo apt install driverctl
sudo driverctl set-override 0000:01:00.0 vfio-pci
sudo driverctl set-override 0000:01:00.1 vfio-pci
```

### Load VFIO & VSOCK modules

[VFIO](https://docs.kernel.org/driver-api/vfio.html) provides framework for virtual machines to utilize host machine hardware directly, allowing high performance device access.

In Gevulot this is needed for direct access to GPU.

**/etc/modules-load.d/vfio.conf**
```
kvmgt
vfio-pci
vfio-iommu-type1
vfio-mdev
```

[VSOCK](https://nanovms.com/dev/tutorials/what-is-vsock-why-use-with-unikernels) provides high speed communication channel between Gevulot node and program running in VM.

**/etc/modules-load.d/vsock.conf**
```
vsock
vsock_vhost
```

### Configure GPU's PCI bus devices to be bound with VFIO 

Depending on specific construction of the GPU device card, there can be multiple PCI device entries on the bus.

In order to make PCI passthrough work with virtual machines, all devices present in PCI bus must be bound with VFIO driver.

Find out the device IDs of the GPU:
```
lspci -nnD | grep -i nvidia
```

**Example output:**
```
0000:01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD102 [GeForce RTX 4090] [10de:2684] (rev a1)
0000:01:00.1 Audio device [0403]: NVIDIA Corporation AD102 High Definition Audio Controller [10de:22ba] (rev a1)
```

In the above example output the first entry on the line is the PCI slot name. The second last column is the PCI ID.

Configure all PCI device IDs to VFIO:

**/etc/modprobe.d/vfio.conf**
```
options vfio-pci ids=10de:2684,10de:22ba
```


### Adjust VFIO device permissions with udev rule

By default, the VFIO group devices are only accessible by root. In order to allow Gevulot node be run as non-root, the VFIO group devices' permissions must be adjusted: 

**/etc/udev/rules.d/99-vfio.rules**
```
SUBSYSTEM=="vfio", OWNER="root", GROUP="kvm"
```

That will allow VFIO device access to all users who belong to `kvm` group.

### Further device driver adjustments

In some cases the GPU board might have devices which have drivers directly built into kernel and which cannot be blacklisted due to critical nature.

One such example is a USB controller embedded on GPU board.

Individual PCI device's driver attachment can be fixed in a following way:
```
echo "0000:01:00.2" | sudo tee /sys/bus/pci/drivers/xhci_hcd/unbind
echo "0000:01:00.2" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind
```

The device specific string above is the PCI slot name (first record of `lspci -nnD`).

For persistent effect, create a systemd unit for this.

## Postgres

Gevulot node uses Postgres database for its operation. It has been tested to work with Postgres releases 15 and 16.

### Example systemd unit

Below is an example systemd unit that can be used as a template to run containerized postgres:

**/etc/containers/systemd/gevulot-postgres.container**
```
[Install]
WantedBy=default.target

[Container]
ContainerName=gevulot-postgres

Image=docker.io/library/postgres:16-alpine

Environment=POSTGRES_USER=<username>
Environment=POSTGRES_PASSWORD=<password>
Environment=POSTGRES_DB=gevulot

Network=host
ExposeHostPort=5432

Volume=/var/lib/postgresql/data:/var/lib/postgresql/data:z
```


### Database migration

In order to create initial tables or perform latest migrations, run:
```
podman run -it --network=host quay.io/gevulot/node:latest migrate [--db-url=postgres://<username>:<password>@<dbhost>/gevulot]
```

## Gevulot networking

Gevulot uses networking in following ways:

- JSON-RPC (over HTTP) to provide an end user API for submitting & querying transactions.
- P2P networking to propagate transactions between the nodes.
- HTTP to download data attached to transactions (including program images).

In order to guarantee full functionality, access to all those services must be allowed.

If Gevulot node is behind NAT, one must configure the public address that external nodes can connect to and configure port forwarding in such a way that both, the P2P port and configured HTTP port is accessible from the outside.

Gevulot help

```
podman run -it quay.io/gevulot/node:latest run -h
```

...displays all possible configuration directives. When running the node as a systemd unit, an environment variable based configuration is preferred for easy configuration management.

## Gevulot node storage

Gevulot node requires plenty of storage for program images and their output data.

Create a directory on a volume with enough space.

This guide uses `/var/lib/gevulot`, which is also the default, but it is configurable.

## Gevulot node key

Each Gevulot node requires a keypair for operation. It can be generated with Gevulot node container:
```
podman run -it -v /var/lib/gevulot:/var/lib/gevulot:z quay.io/gevulot/node:latest generate node-key 
```

## Gevulot node systemd unit

Gevulot node is packaged as a container for best possible platform compatibility and it is recommended to run it with Podman Quadlet. 

To pass VFIO devices into container, check the specific devices for your installation and update them to systemd unit below:
```
ls /dev/vfio
```

Use following systemd unit as basis for running Gevulot node:

**/etc/containers/systemd/gevulot-node.container**
```
[Install]
WantedBy=default.target

[Unit]
Requires=gevulot-postgres.service
After=gevulot-postgres.service

[Container]
ContainerName=gevulot-node

Image=quay.io/gevulot/node:latest
AutoUpdate=registry

Environment=RUST_LOG=warn,gevulot=debug,sqlx=error
Environment=GEVULOT_DB_URL=postgres://<user>:<password>@<host>/gevulot
Environment=GEVULOT_GPU_DEVICES=0000:01:00.0
Environment=GEVULOT_P2P_DISCOVERY_ADDR=34.88.251.176:9999
Environment=GEVULOT_PSK_PASSPHRASE="<coordinated pre-shared key for P2P network>"

Network=host

# Expose JSON-RPC.
ExposeHostPort=9944

# Expose HTTP
ExposeHostPort=9995

# Expose P2P.
ExposeHostPort=9999

# Bind VSOCK & GPU into container.
AddDevice=/dev/kvm:rw
AddDevice=/dev/vsock:rw
AddDevice=/dev/vhost-vsock:rw
AddDevice=/dev/vfio/59:rw
AddDevice=/dev/vfio/vfio:rw

# Disable SELinux labelling to allow access to VFIO devices.
SecurityLabelDisable=true

# Allow larger memlock limit for QEMU / VFIO use
PodmanArgs=--ulimit=memlock=-1:-1

# Mount host directory for Gevulot files.
Volume=/var/lib/gevulot:/var/lib/gevulot:z
```

## Auto-update

In order to receive automatic updates for Gevulot node, enable `podman-auto-update` timer:
```
sudo systemctl enable podman-auto-update.timer
```

This will update the container registry every night.

## Manual update

To manually update containers at will, run following:
```
sudo systemctl start podman-auto-update
```
