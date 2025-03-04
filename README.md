# Metal3 Ironic Container

This repo contains the files needed to build the Ironic images used by Metal3.

## Build Status

[![Ubuntu daily main build status](https://jenkins.nordix.org/buildStatus/icon?job=metal3_daily_main_integration_test_ubuntu&subject=Ubuntu%20daily%20main)](https://jenkins.nordix.org/view/Metal3/job/metal3_daily_main_integration_test_ubuntu/)
[![CentOS daily main build status](https://jenkins.nordix.org/buildStatus/icon?job=metal3_daily_main_integration_test_centos&subject=CentOS%20daily%20main)](https://jenkins.nordix.org/view/Metal3/job/metal3_daily_main_integration_test_centos/)

## Description

When updated, builds are automatically triggered on
<https://quay.io/repository/metal3-io/ironic/>

This repo supports the creation of multiple containers needed when provisioning
baremetal nodes with Ironic. Eventually there will be separate images for each
container, but currently separate containers can share this same image with
specific entry points.

The following entry points are provided:

- `runironic` - Starts the ironic-conductor and ironic-api processes to manage
   the provisioning of baremetal nodes.  Details on Ironic can be found at
   <https://docs.openstack.org/ironic/latest/>.  This is the default entry point
   used by the Dockerfile.
- `rundnsmasq` - Runs the dnmasq dhcp server to provide addresses and initiate
   PXE boot of baremetal nodes.  This includes a lightweight TFTP server.
   Details on dnsmasq can be found at
   <http://www.thekelleys.org.uk/dnsmasq/doc.html>.
- `runhttpd` - Starts the Apache web server to provide images via http for PXE
   boot and for deployment of the final images.
- `runlogwatch` - Waits for host provisioning ramdisk logs to appear, prints
   their contents and deletes files.

All of the containers must share a common mount point or data store.  Ironic
requires files for both the TFTP server and HTTP server to be stored in the same
partition.  This common store must include, in `<shared store>/html/images`,
the following images:

- ironic-python-agent.kernel
- ironic-python-agent.initramfs
- final image to be deployed onto node in qcow2 format

The following environment variables can be passed in to customize run-time
functionality:

- `PROVISIONING_MACS` - a comma seperated list of mac address of the master
   nodes (used to determine the `PROVISIONING_INTERFACE`)
- `PROVISIONING_INTERFACE` - interface to use for ironic, dnsmasq(dhcpd) and
   httpd (default provisioning, this is calculated if the above
   `PROVISIONING_MACS` is provided)
- `DNSMASQ_EXCEPT_INTERFACE` - interfaces to exclude when providing DHCP address
  (default `lo`)
- `HTTP_PORT` - port used by http server (default `80`)
- `DHCP_RANGE` - dhcp range to use for provisioning (default
   `172.22.0.10-172.22.0.100`)
- `DHCP_HOSTS` - a `;` separated list of `dhcp-host` entries, e.g. known MAC
   addresses like `00:20:e0:3b:13:af;00:20:e0:3b:14:af` (empty by default). For
   more details on `dhcp-host` see
   [the man page](https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html).
- `DHCP_IGNORE` - a set of tags on hosts that should be ignored and not allocate
   DHCP leases for, e.g. `tag:!known` to ignore any unknown hosts (empty by
   default)
- `MARIADB_PASSWORD` - The database password
- `OS_<section>_\_<name>=<value>` - This format can be used to set arbitary
   Ironic config options
- `IRONIC_RAMDISK_SSH_KEY` - A public key to allow ssh access to nodes running
   IPA, takes the format "ssh-rsa AAAAB3....."
- `IRONIC_KERNEL_PARAMS` - This parameter can be used to add additional kernel
   parameters to nodes running IPA
- `GATEWAY_IP` - gateway IP address to use for ironic dnsmasq(dhcpd)
- `DNS_IP` - DNS IP address to use for ironic dnsmasq(dhcpd)

The ironic configuration can be overridden by various environment variables.
The following can serve as an example:

- `OS_CONDUCTOR__DEPLOY_CALLBACK_TIMEOUT=4800` - timeout (seconds) to wait for
   a callback from a deploy ramdisk
- `OS_CONDUCTOR__INSPECT_TIMEOUT=1800` - timeout (seconds) for waiting for node
   inspection
- `OS_CONDUCTOR__CLEAN_CALLBACK_TIMEOUT=1800` - timeout (seconds) to wait for a
   callback from the ramdisk doing the cleaning
- `OS_PXE__BOOT_RETRY_TIMEOUT=1200` - timeout (seconds) to enable boot retries.

## Build Ironic Image from source

Normally the ironic image is built using RPMs coming from the RDO project.
It is possible to build it using source code setting the **INSTALL_TYPE**
argument to **source** at build time; for example:

```bash
podman build -t ironic-image -f Dockerfile --build-arg INSTALL_TYPE=source
```

When building the ironic image from source, it is also possible to specify a
different source for ironic, ironic-inspector or the sushy library using the
build arguments **IRONIC_SOURCE**, **IRONIC_INSPECTOR_SOURCE**, and
**SUSHY_SOURCE**. The accepted formats are gerrit refs, like
_refs/changes/89/860689/2_, commit hashes, like
_a1fe6cb41e6f0a1ed0a43ba5e17745714f206f1f_, or a local directory
that needs to be under the sources/ directory in the container context.
An example of a full command installing ironic from a gerrit patch is:

```bash
podman build -t ironic-image -f Dockerfile --build-arg INSTALL_TYPE=source \
    --build-arg IRONIC_SOURCE="refs/changes/89/860689/2"
```

An example using the local directory _sources/ironic_:

```bash
podman build -t ironic-image -f Dockerfile --build-arg INSTALL_TYPE=source \
    --build-arg IRONIC_SOURCE="ironic"
```

## Apply project patches to the images during build

When building the image, it is possible to specify a patch of one or more
upstream projects to apply to the image using the **PATCH_LIST** argument in
the cli command, for example:

```bash
podman build -t ironic-image -f Dockerfile --build-arg \
    PATCH_LIST=my-patch-list
```

The **PATCH_LIST** argument is a path to a file under the image context.
Its format is a simple text file that contains references to upstream patches
for the ironic projects.
Each line of the file is in the form:
    **project_dir refspec (git_host)**
where:

- **project_dir** is the last part of the project url including the
  organization, for example for ironic is _openstack/ironic_
- **refspec** is the gerrit refspec of the patch we want to test, for example if
  you want to apply the patch at
  <https://review.opendev.org/c/openstack/ironic/+/800084>
  the refspec will be _refs/changes/84/800084/22_
  Using multiple refspecs is convenient in case we need to test patches that
  are connected to each other, either on the same project or on different
  projects.
- **git_host** (optional) is the git host from which the project will be cloned.
  If unset, `https://opendev.org` is used.
