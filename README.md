# Cloud-Init Datasource for VMware GuestInfo
This project provides a cloud-init datasource for pulling meta,
user, and vendor data from VMware vSphere's GuestInfo [interface](https://github.com/vmware/govmomi/blob/master/govc/USAGE.md#vmchange).

### Goal of this release
the lastest cloud-init expect user-data and vendor-data encapsuled inside a mime message.

### Installing on Ubuntu bionic
The VMware GuestInfo datasource can be installed on any Linux distribution
where cloud-init is already present. To do so, simply execute the following:

```shell
$ curl -sSL https://raw.githubusercontent.com/Fred78290/cloud-init-vmware-guestinfo/master/install.sh | sh -
```

At the end, the VM will shutdown after calling cloud-init clean.

## Configuration
The data source is configured by setting `guestinfo` properties on a 
VM's `extraconfig` data or a customizable vApp's `properties` data.

| Property | Description |
|----------|-------------|
| `guestinfo.metadata` | A JSON string containing the cloud-init metadata. |
| `guestinfo.metadata.encoding` | The encoding type for `guestinfo.metadata`. |
| `guestinfo.userdata` | A YAML document containing the cloud-init user data. |
| `guestinfo.userdata.encoding` | The encoding type for `guestinfo.userdata`. |
| `guestinfo.vendordata` | A YAML document containing the cloud-init vendor data. |
| `guestinfo.vendordata.encoding` | The encoding type for `guestinfo.vendordata`. |

All `guestinfo.*.encoding` property values may be set to `base64` or 
`gzip+base64`.

## Walkthrough
The following series of steps is a demonstration on how to configure a VM
with cloud-init and the VMX GuestInfo datasource.

### Create a network configuration file
First, create the network configuration for the VM. Save the following 
YAML to a file named `network.config.yaml`:

```yaml
version: 1
config:
  - type: physical
    name: ens192
    subnets:
      - type: static
        address: 192.168.1.200
        gateway: 192.168.1.1
        dns_nameservers:
          - 8.8.8.8
          - 8.8.4.4
        dns_search:
          - vmware.ci
```

See the section on [configuring the network](#configuring-the-network) for
more information on the network configuration schema.

### Create a metadata file
Next, create a JSON file named `metadata.json`:

```json
{
  "network": "NETWORK_CONFIG",
  "network.encoding": "gzip+base64",
  "local-hostname": "cloud-vm",
  "instance-id": "cloud-vm"
}
```

Please note that in addition to the `network` key in the metadata there
is also a key named `network.encoding`. This key informs the datasource
how to decode the `network` data. Valid values for `network.encoding`
include:

* `base64`
* `gzip+base64`

### Create a cloud-config file
Finally, create the cloud-config file `cloud-config.yaml`:

```yaml
#cloud-config

users:
  - default
  - name: akutz
    primary_group: <USERNAME>
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: sudo, wheel
    ssh_import_id: None
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-rsa <SSH PUB KEY>
```

### Assigning the cloud-config data to the VM's GuestInfo
Please note that this step requires that the VM be powered off. All of
the commands below use the VMware CLI tool, 
[`govc`](https://github.com/vmware/govmomi/blob/master/govc).

Go ahead and assign the path to the VM to the environment variable `VM`:
```shell
$ export VM="/inventory/path/to/the/vm"
```

Next, power off the VM:
```shell
$ govc vm.power -off "${VM}"
```

Export the environment variables that contain the cloud-init metadata
and cloud-config:
```shell
$ export CLOUD_CONFIG=$(gzip -c9 <cloud-config.yaml | base64)
$ export METADATA=$(sed 's~NETWORK_CONFIG~'"$(gzip -c9 <network-config.yaml | \
                    base64)"'~' <metadata.json | gzip -9 | base64)
```

Assign the metadata and cloud-config to the VM's extra configuration
dictionary, `guestinfo`:
```shell
$ govc vm.change -vm "${VM}" -e guestinfo.metadata="${METADATA}"
$ govc vm.change -vm "${VM}" -e guestinfo.metadata.encoding=gzip+base64
$ govc vm.change -vm "${VM}" -e guestinfo.userdata="${CLOUD_CONFIG}"
$ govc vm.change -vm "${VM}" -e guestinfo.userdata.encoding=gzip+base64
```

Please note the above commands include specifying the encoding for the
properties. This is important as it informs the datasource how to decode
the data for cloud-init. Valid values for `metadata.encoding` and
`userdata.encoding` include:

* `base64`
* `gzip+base64`

### Using the cloud-init VMX GuestInfo datasource
Power the VM back on.
```shell
$ govc vm.power -vm "${VM}" -on
``` 

If all went according to plan, the CentOS box is:
* Locked down, allosing SSH access only for the user in the cloud-config
* Configured for a static IP address, 192.168.1.200
* Has a hostname of `centos-cloud`

## Examples
This section reviews common configurations:

### Setting the hostname
The hostname is set by way of the metadata key `local-hostname`.

### Setting the instance ID
The instance ID may be set by way of the metadata key `instance-id`.
However, if this value is absent then then the instance ID is
read from the file `/sys/class/dmi/id/product_uuid`.

### Configuring the network
The network is configured by setting the metadata key `network`
with a value consistent with Network Config Versions 
[1](http://bit.ly/cloudinit-net-conf-v1) or 
[2](http://bit.ly/cloudinit-net-conf-v2),
depending on the Linux distro's version of cloud-init.

The metadata key `network.encoding` may be used to indicate the
format of the metadata key "network". Valid encodings are `base64`
and `gzip+base64`.

## Conclusion
To learn more about how to use cloud-init with CentOS, please see the cloud-init
[documentation](https://cloudinit.readthedocs.io/en/latest/index.html) for more 
examples and reference information for the cloud-config files.
