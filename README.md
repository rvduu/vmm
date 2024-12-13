# VMM - Virtual Machine Manager
Wrapper script to control the creation and cloning of **Red Hat Enterprise Linux (RHEL)** template VMs using virsh / virt-install / virt-sysprep.

### Requirements

This script is based and tested on a Fedora host (other distro may also work) and requires:

1. sudo access to run virsh / virt-install / virt-sysprep / qemu-img / mv / cp
1. A Working `libvirt` + `qemu-kvm` setup and some additional packages

        $ sudo dnf install libvirt guestfs-tools, libvirt-nss

1. Enabled `libvirt` NSS module to allow connecting to VMs using the hostname or libivirt domain name
    1. Check current `authselect` profile and the enabled features (see also `/etc/authselect/authselect.conf`)

            $ sudo authselect current
            Profile ID: sssd
            Enabled features:
            - with-silent-lastlog
            - with-fingerprint

    1. Update the `authselect` profile using the already configured features and include `with-libvirt`

            $ sudo authselect select sssd with-silent-lastlog with-fingerprint with-libvirt

1. Access to run `virsh` as a regular user using Polkit (see also https://gist.github.com/rvduu/4a07faa1cb1c507b0ef7e16580bc239d)
1. Red Hat Enterprise Linux installation DVD iso's for the templates, downloaded from http://access.redhat.com to `/var/lib/libvirt/images`
1. A Working Activation key (Create one at https://console.redhat.com/insights/connector/activation-keys)

### Usage

## Creating a template

With a default Fedora host with libvirt installed, this script uses the `default` storage pool for hosting the VM disk and iso images as well the `default` network. The kickstart templates are installing template VMs that can later be copied to easily and quickly create a new VM. The base image is by default named `rhel{x.y}base` and will be sealed to remove all config using `virt-sysprep`. Upon cloning a VM, the base image is copied, a VM created and will be configured using `cloud-init`.

The template VM are not especially secure and thus should **not** be used to production workload unless secured/locked afterwards. Core details of the template VM:
* Default install as a "Server" image
* Auto partition with "plain" partitions for easy filesystem resize (grow)
* Default domainname `example.com` (network should have a entry with `<domain name='example.com'/>` in the xml definition)
* Serial console enabled (accessible via `virsh console <domain>`)
* The cloud-init will configure the `admin` user without password (locked)
* Default SSH pub key's of the local user configured for the `admin` user

After kickstarting, the template VM's disk is sparcified to alllow minimal disk usage and fast cloning times.

Example kickstarting a RHEL8.8 template VM (run `vmm help` for all options):

    $ vmm createbase 8.8 


## Creating a new VM by cloning a template

Upon cloning, the base image is copied and imported using `virt-install` and will be configured usgin `cloud-init`.

Example:

    $ vmm clone 8.8 test88

To specify non-default memory and vCPU options:

    $ vmm clone -m 8 -c 4 8.8 test88

To resize the disk, root partition and root filesystem, us the -R option:

    $ vmm clone -R "+10G" 8.8 test88

To register the host against the Red Hat Customer Portal using an activation key (username and password is also possibe) and update all packages to the latest version (mutually exclusive with the -i option):

    $ vmm clone -r -a activation_key -o orgid -U 8.8 test88

Instead of registring the host to the Red Hat Customer Portal, it is also possible to attach the Installation ISO and configure yum/dnf to use the packages from the ISO (mutually exclusive with the -r option):

    $ vmm clone -i 8.8 test88

See `vmm help` for all available options.


## Accessing a VM

The base images and cloned images are created using the `default` network in `libvirt`. This network is by default a NATted network. By using `libvirt-nss`, the Name Service Switch will utilize the DHCP lease info from the `libvirt` host to resolve the VMs IP number when using `gethostbyname(3)`. See the 'Requirements' section on how to enable `libvirt-nss`.

This default network is often used for testing purposed and to avoid filling up `$HOME/.ssh/known_hosts` with host keys of short lived VMs, the `vmm` script has one extra command to easily connect to the VM using `ssh` and will disable the StrictHostKey checking by adding the folloing options to the `ssh` command line (warning: this can be insecure):

    -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null

For example, to connect to a VM:

    vmm connect test88

## Customization by setting default values for variables

All variables in the top section of the script (inclusing the Kickstart templates) can be overriden by stroring them in `$HOME/.vvm.conf`. Because it can contain activation keys or login information, this file should not be readable by others (`chmod 0600 $HOME/.vmm.conf`).

It is also possible to use a custom kickstart template. Dump the default kickstart template with:

    vmm dumpks 8.8 > customkstemplate.cfg

Use the custom kickstart template to create a new base image with:

    vmm createbase -t customkstemplate.cfg 8.8 custom88

Or store the template as multline text in the `KS{7,8,9}TEMPLATE` variable in `$HOME/.vmm.conf`.
