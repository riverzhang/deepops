# DeepOps Virtual

Set up a virtual cluster with DeepOps. Useful for...

1. Learning how to deploy DeepOps on limited hardware
2. Testing new features in DeepOps
3. Tailoring DeepOps in a local environment before deploying it to the production cluster

## Requirements

### Hardware Requirements

The host machine should have enough resources to fufill the minimum VM needs...

Total: 6 vCPU, 20 GB RAM, 96 GB Storage
* virtual-login: 2 vCPU, 2GB RAM and 32GB Storage
* virtual-mgmt: 2 vCPU, 2GB RAM and 32GB Storage
* virtual-gpu01: 2 vCPU, 16GB RAM and 32GB Storage

### Operating System Requirements

* Ubuntu 16.04 (or greater)
* CentOS 7.6 (or greater)

Running DeepOps virtually assumes that the host machine's OS is an approved OS. If this is not the case, the scripts used in the steps below may be modified to work with a different OS.

Also, using VMs and optionally GPU passthrough assumes that the host machine has been configured to enable virtualization in the BIOS. For instructions on how to accomplish this, refer to the sections at the bottom of this README: [Enabling virtualization and GPU passthrough](#enabling-virtualization-and-gpu-passthrough).

## Start the Virtual Cluster

1. From the main deepops directory, run the setup script.

   This will install Ansible and other software on the provisioning machine which will be used to deploy all other software to the cluster. For more information on Ansible and why we use it, consult the [Ansible Guide](/docs/ANSIBLE.md).

   ```sh
   ./scripts/setup.sh
   ```

2. In the virtual directory, startup vagrant. This will start 3 VMs by default.

   ```sh
   # NOTE: The default VM OS is Ubuntu. If you wish the VMs to spawn CentOS,
   #       configure the DEEPOPS_VAGRANT_FILE variable accordingly...
   #       export DEEPOPS_VAGRANT_FILE=$(pwd)/Vagrantfile-centos
   # NOTE: virtual-gpu01 requires GPU passthrough, by default it is not enabled
   # NOTE: 3 VMs are started by default: virtual-mgmt, virtual-login, virtual-gpu01
   
   cd virtual
   ./vagrant_startup.sh
   ```

3. Start the cluster.

   ```sh
   # NOTE: Only Kubernetes is deployed by default. To also deploy Slurm,
   #       configure the DEEPOPS_ENABLE_SLURM variable accordingly...
   #       export DEEPOPS_ENABLE_SLURM=1
   
   ./cluster_up.sh
   ```
   
   This script will run the ansible playbooks to deploy DeepOps to the Vagrant VMs and should complete without errors.
   
4. Set up the Kubernetes environment.

   As part of `cluster_up.sh`, a fresh `kubectl` executable and the Kubernetes cluster's `admin.conf` are downloaded. To use these so commands may be run locally, a convenient script may be sourced...
   
   ```sh
   source k8s_environment.sh
   ```
   
   Optionally, `kubectl` can permanently be added to the PATH and `admin.conf` can be copied to `~/.kube/config`, which results in a more permanent solution.

## Using the Virtual Cluster

### Kubernetes

Consult the [Kubernetes Usage Guide](/docs/kubernetes-usage.md) for examples of how to use Kubernetes.

### Connecting to the VMs

Connect to any of the VM nodes directly via vagrant ssh...

```sh
# NOTE: Must be in the `deepops/virtual` directory

vagrant ssh virtual-gpu01
```

## Destroy the Virtual Cluster

To destroy the cluster and shutdown the VMs, run the `vagrant_shutdown.sh` script...

```sh
./vagrant_shutdown.sh
```

Check that there are no running VMs using `virsh list`...

```sh
$ virsh list --all
 Id    Name                           State
----------------------------------------------------
```

## Configure GPU passthrough

If the host machine has a GPU and is configured for GPU passthrough, it is possible to configure the `virtual-gpu01` VM to use the GPU.

Run `lspci` to discover the appropriate bus...

```sh
$ lspci -nnk | grep NVIDIA
07:00.0 VGA compatible controller [0300]: NVIDIA Corporation Device [10de:15fc] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1195]
07:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:0fb1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1195]
08:00.0 VGA compatible controller [0300]: NVIDIA Corporation Device [10de:15fc] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1195]
08:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:0fb1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1195]
0e:00.0 VGA compatible controller [0300]: NVIDIA Corporation Device [10de:15fc] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1195]
0e:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:0fb1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1195]
0f:00.0 VGA compatible controller [0300]: NVIDIA Corporation Device [10de:15fc] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1195]
0f:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:0fb1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1195]
```

In this example, the GPU at `08:00.0` is chosen.

In the `Vagrantfile`, uncomment the `v.pci` configuration and update it with a mapping to the bus discovered with `lspci`...

```
v.pci :bus => '0x08', :slot => '0x00', :function => '0x0'
```

Note that more than one GPU may be passed through by adding additional `v.pci` lines.

Next, shutdown the virtual cluster (if it is running) and startup vagrant + run cluster up again...

```sh
./vagrant_shutdown.sh
./vagrant_startup.sh
./cluster_up.sh
```

# Enabling virtualization and GPU passthrough

On many machines, virtualization and GPU passthrough are not enabled by default. Follow these directions so that a virtual DeepOps cluster can start on your host machine with GPU access on the VMs.

## BIOS and bootloader changes

To support KVM, we need GPU pass through. To enable GPU pass through, we need to enable VFIO support in BIOS and Bootloader.

### BIOS changes

* Enable BIOS settings: Intel VT-d and Intel VT-x
* Enable BIOS support for large-BAR1 GPUs: 'MMIO above 4G' or 'Above 4G encoding', etc.

**DGX-2 SBIOS**
* VT-x:  enable
* VT-d:  enable
* MMIO above 4G: enable
* Intel AMT: disable

**DGX-1/1V SBIOS**
* VT-x: Intel RC Setup -> Processor Configuration -> VMX
* VT-d: Intel RC Setup -> IIO Configuration -> VT-d
* MMIO above 4G: Advanced -> PCI Subsystem Setting -> Above 4G Encoding

**DGX Station SBIOS**
* VT-x: 
* VT-d: 
* MMIO above 4G: verify virtualization support is enabled in the BIOS, by looking for vmx for Intel or svm for AMD processors...

```
$ grep -oE 'svm|vmx' /proc/cpuinfo | uniq
vmx
```

### Bootloader changes

1. Add components necessary to load VFIO (Virtual Function I/O). VFIO is required to pass full devices through to a virtual machine, so that Ubuntu loads everything it needs. Edit and add the following to `/etc/modules` file:
```
pci_stub
vfio
vfio_iommu_type1
vfio_pci
kvm
kvm_intel
```

2. Next, need Ubuntu to load IOMMU properly. Edit `/etc/default/grub` and modify "GRUB_CMDLINE_LINUX_DEFAULT", by adding "intel_iommu=on" to enable IOMMU. May also need to add "vfio_iommu_type1.allow_unsafe_interrupts=1" if interrupt remapping should be enabled. Post these changes, the GRUB command line should look like this:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1
iommu=pt"
```

3. Enable the vfio-pci driver on boot:
```
$ echo vfio-pci | sudo tee /etc/modules-load.d/vfio-pci.conf
```

4. Run `sudo update-grub` to update GRUB with the new settings and reboot the system.

### Blacklist the GPU devices

We do not want the host running DGX Base OS to use the GPU Devices. Instead we want Guest VMs to get full access to the NVIDIA GPU devices. Hence, in the DGX Base OS on the host,  blacklist them by adding their IDs to the initramfs.

1. Run the command `lspci -nn | grep NVIDIA` to get the list of PCI-IDs
```
08:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
0a:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
10:00.0 Bridge [0680]: NVIDIA Corporation Device [10de:1ac1] (rev a1)
11:00.0 Bridge [0680]: NVIDIA Corporation Device [10de:1ac1] (rev a1)
12:00.0 Bridge [0680]: NVIDIA Corporation Device [10de:1ac1] (rev a1)
18:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
1a:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
89:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
8b:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
92:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
94:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
```

2. Edit `/etc/modprobe.d/vfio.conf` and add this line:
```
options vfio-pci ids=10de:1db1,10de:1ac1
```

NOTE: First entry is for Volta and the latter for NVSwitch

3. Rebuild the initramfs by running `sudo update-initramfs -u` and reboot the system.

4. After the system reboots, verify GPU devices and NVSwitches are claimed by vfio_pci driver by running `dmesg | grep vfio_pci`...
```
[   15.668150] vfio_pci: add [10de:1db1[ffff:ffff]] class 0x000000/00000000
[   15.736099] vfio_pci: add [10de:1ac1[ffff:ffff]] class 0x000000/00000000
```

```
$ lspci -nnk -d 10de:1ac1
10:00.0 Bridge [0680]: NVIDIA Corporation Device [10de:1ac1] (rev a1)
	Kernel driver in use: vfio-pci
11:00.0 Bridge [0680]: NVIDIA Corporation Device [10de:1ac1] (rev a1)
	Kernel driver in use: vfio-pci
12:00.0 Bridge [0680]: NVIDIA Corporation Device [10de:1ac1] (rev a1)
	Kernel driver in use: vfio-pci
```

If the `Kernel driver in use` is not `vfio-pci` and instead the nvidia driver, it may be necessary to blacklist the nvidia driver or instruct it to load vfio-pci beforehand...

```
$ cat /etc/modprobe.d/nvidia.conf
softdep nvidia_384 pre: vfio-pci
```

One more check...

```
$ lspci -nnk -d 10de:1db1
08:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1212]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia_vgpu_vfio, nvidia
0a:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1212]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia_vgpu_vfio, nvidia
18:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1212]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia_vgpu_vfio, nvidia
1a:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1212]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia_vgpu_vfio, nvidia
89:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1212]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia_vgpu_vfio, nvidia
8b:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1212]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia_vgpu_vfio, nvidia
92:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1212]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia_vgpu_vfio, nvidia
94:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:1db1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1212]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia_vgpu_vfio, nvidia
```



