# Firecracker

Firecracker is an open source virtualization technology that is purpose-built for creating and managing secure, multi-tenant container and function-based services  that provide serverless operational models. Firecracker runs workloads in lightweight virtual machines, called microVMs, which combine the security and isolation properties provided by hardware virtualization technology with the speed and flexibility of containers. It is written in Rust, a modern programming language that guarantees thread safety and prevents many types of buffer overrun errors that can lead to security vulnerabilities.

Firecracker was developed at Amazon Web Services to improve the customer experience of services like AWS Lambda and AWS Fargate.
(Extra: Both Lambda and Fargate are serverless services. AWS Lambda charges you per invocation and duration of each invocation whereas AWS Fargate charges you for the vCPU and memory resources of your containerized applications use per second.)

Firecracker is a virtual machine monitor (VMM) that uses the Linux Kernel-based Virtual Machine (KVM) to create and manage microVMs. Firecracker has a minimalist design. It excludes unnecessary devices and guest functionality to reduce the memory footprint and attack surface area of each microVM. This improves security, decreases the startup time, and increases hardware utilization. Firecracker is generally available on 64-bit Intel, AMD and Arm CPUs with support for hardware virtualization.

## How It Works?
![image](https://user-images.githubusercontent.com/54535154/150671385-fb3e4560-b721-4378-a87e-a79e3bccdac0.png)

Firecracker runs in user space and uses the Linux Kernel-based Virtual Machine (KVM) to create microVMs. The fast startup time and low memory overhead of each microVM enables you to pack thousands of microVMs onto the same machine. This means that every function, container, or container group can be encapsulated with a virtual machine barrier, enabling workloads from different customers to run on the same machine, without any tradeoffs to security or efficiency. Firecracker is an alternative to QEMU , an established VMM with a general purpose and broad feature set that allows it to host a variety of guest operating systems.

You can control the Firecracker process via a RESTful API that enables common actions such as configuring the number of vCPUs or starting the machine. It provides built-in rate limiters, which allows you to granularly control network and storage resources used by thousands of microVMs on the same machine. You can create and configure rate limiters via the Firecracker API and define flexible rate limiters that support bursts or specific bandwidth/operations limitations. Firecracker also provides a metadata service that securely shares configuration information between the host and guest operating system. You can set up and configure the metadata service using the Firecracker API. Each Firecracker microVM is further isolated with common Linux user-space security barriers by a companion program called "jailer". The jailer provides a second line of defense in case the virtualization barrier is ever compromised.

## Host Integration

The following diagram depicts an example host running Firecracker microVMs.

![image](https://user-images.githubusercontent.com/54535154/150674615-691b1d80-eca4-4b11-9399-da03711de8bc.png)

Firecracker runs on Linux hosts and with Linux guest OSs.

## Storage

Firecracker emulated block devices are backed by files on the host. To be able to mount block devices in the guest, the backing files need to be pre-formatted with a filesystem that the guest kernel supports.

## Internal Architecture

Each Firecracker process encapsulates one and only one microVM. The process runs the following threads: API, VMM and vCPU(s). The API thread is responsible for Firecracker's API server and associated control plane. It's never in the fast path of the virtual machine. The VMM thread exposes the machine model, minimal legacy device model, microVM metadata service (MMDS) and VirtIO device emulated Net, Block and Vsock devices, complete with I/O rate limiting. In addition to them, there are one or more vCPU threads (one per guest CPU core). They are created via KVM and run the KVM_RUN main loop. They execute synchronous I/O and memory-mapped I/O operations on devices models.

## Threat Containment

From a security perspective, all vCPU threads are considered to be running malicious code as soon as they have been started; these malicious threads need to be contained. Containment is achieved by nesting several trust zones which increment from least trusted or least safe (guest vCPU threads) to most trusted or safest (host). These trusted zones are separated by barriers that enforce aspects of Firecracker security. For example, all outbound network traffic data is copied by the Firecracker I/O thread from the emulated network interface to the backing host TAP device, and I/O rate limiting is applied at this point. These barriers are marked in the diagram below.

![image](https://user-images.githubusercontent.com/54535154/150674937-8ef50599-da61-4274-bafc-28e9a3dc8d80.png)

## Security aspects

1. Simple Guest Model – Firecracker guests are presented with a very simple virtualized device model in order to minimize the attack surface: a network device, a block I/O device, a Programmable Interval Timer, the KVM clock, a serial console, and a partial keyboard (just enough to allow the VM to be reset).
2. Process Jail – The Firecracker process is jailed using cgroups and seccomp BPF, and has access to a small, tightly controlled list of system calls.
3. Static Linking – The firecracker process is statically linked, and can be launched from a jailer to ensure that the host environment is as safe and clean as possible.

## Benefits

1. Security from the ground up: 
Firecracker microVMs use KVM-based virtualizations that provide enhanced security over traditional VMs. This ensures that workloads from different end customers can run safely on the same machine. Firecracker also implements a minimal device model that excludes all non-essential functionality and reduces the attack surface area of the microVM.
2. Speed by design: 
In addition to a minimal device model, Firecracker also accelerates kernel loading and provides a minimal guest kernel configuration. This enables fast startup times. Firecracker initiates user space or application code in as little as 125 ms and supports microVM creation rates of up to 150 microVMs per second per host.
3. Scale and efficiency: 
Each Firecracker microVM runs with a reduced memory overhead of less than 5 MiB, enabling a high density of microVMs to be packed on each server. Firecracker provides a rate limiter built into every microVM. This enables optimized sharing of network and storage resources, even across thousands of microVMs.

## Getting hands dirty

Execute the below command to use firecracker from a regular user.
```console
sudo setfacl -m u:${USER}:rw /dev/kvm
```
Firecracker works best on 4.14 and 5.10 kernel versions. Check yours
```console
uname -r
```
Do the KVM setup as stated [here](https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md#appendix-a-setting-up-kvm-access)

Get the latest binary from [here](https://github.com/firecracker-microvm/firecracker/releases)

Rename the binary to "firecracker":
```console
mv firecracker-${latest}-$(uname -m) firecracker
```
Get an uncompressed Linux kernel binary, and an ext4 file system image (to use as rootfs).
1. To run an `x86_64` guest you can download such resources from:
    [kernel](https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/x86_64/kernels/vmlinux.bin)
    and [rootfs](https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/x86_64/rootfs/bionic.rootfs.ext4).
2. To run an `aarch64` guest, download them from:
    [kernel](https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/aarch64/kernels/vmlinux.bin)
    and [rootfs](https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/aarch64/rootfs/bionic.rootfs.ext4).
    
Now, let's open up two shell prompts: one to run Firecracker, and another one
to control it (by writing to the API socket). For the purpose of this guide,
**make sure the two shells run in the same directory where you placed the
`firecracker` binary**.

Change the access permissions of the binary
```console
chmod 777 firecracker
```

In your **first shell**:

- make sure Firecracker can create its API socket:

```bash
rm -f /tmp/firecracker.socket
```

- then, start Firecracker:

```bash
./firecracker --api-sock /tmp/firecracker.socket
```

In your **second shell** prompt:

- get the kernel and rootfs, if you don't have any available:

  ```bash
  arch=`uname -m`
  dest_kernel="hello-vmlinux.bin"
  dest_rootfs="hello-rootfs.ext4"
  image_bucket_url="https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/$arch"

  if [ ${arch} = "x86_64" ]; then
      kernel="${image_bucket_url}/kernels/vmlinux.bin"
      rootfs="${image_bucket_url}/rootfs/bionic.rootfs.ext4"
  elif [ ${arch} = "aarch64" ]; then
      kernel="${image_bucket_url}/kernels/vmlinux.bin"
      rootfs="${image_bucket_url}/rootfs/bionic.rootfs.ext4"
  else
      echo "Cannot run firecracker on $arch architecture!"
      exit 1
  fi

  echo "Downloading $kernel..."
  curl -fsSL -o $dest_kernel $kernel

  echo "Downloading $rootfs..."
  curl -fsSL -o $dest_rootfs $rootfs

  echo "Saved kernel file to $dest_kernel and root block device to $dest_rootfs."
  ```

- set the guest kernel (assuming you are in the same directory as the
  above script was run):

  ```bash
  arch=`uname -m`
  kernel_path=$(pwd)"/hello-vmlinux.bin"

  if [ ${arch} = "x86_64" ]; then
      curl --unix-socket /tmp/firecracker.socket -i \
        -X PUT 'http://localhost/boot-source'   \
        -H 'Accept: application/json'           \
        -H 'Content-Type: application/json'     \
        -d "{
              \"kernel_image_path\": \"${kernel_path}\",
              \"boot_args\": \"console=ttyS0 reboot=k panic=1 pci=off\"
         }"
  elif [ ${arch} = "aarch64" ]; then
      curl --unix-socket /tmp/firecracker.socket -i \
        -X PUT 'http://localhost/boot-source'   \
        -H 'Accept: application/json'           \
        -H 'Content-Type: application/json'     \
        -d "{
              \"kernel_image_path\": \"${kernel_path}\",
              \"boot_args\": \"keep_bootcon console=ttyS0 reboot=k panic=1 pci=off\"
         }"
  else
      echo "Cannot run firecracker on $arch architecture!"
      exit 1
  fi
  ```

- set the guest rootfs:

  ```bash
  rootfs_path=$(pwd)"/hello-rootfs.ext4"
  curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT 'http://localhost/drives/rootfs' \
    -H 'Accept: application/json'           \
    -H 'Content-Type: application/json'     \
    -d "{
          \"drive_id\": \"rootfs\",
          \"path_on_host\": \"${rootfs_path}\",
          \"is_root_device\": true,
          \"is_read_only\": false
     }"
  ```

- start the guest machine:

  ```bash
  curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT 'http://localhost/actions'       \
    -H  'Accept: application/json'          \
    -H  'Content-Type: application/json'    \
    -d '{
        "action_type": "InstanceStart"
     }'
  ```
  ## Voila!
  We have got our microVM up and running. Go ahead and have fun!
  
  ```console
  Ubuntu 18.04.5 LTS ubuntu-fc-uvm ttyS0

  ubuntu-fc-uvm login: root (automatic login)

  Last login: Sun Jan 23 07:35:59 UTC 2022 on ttyS0
  Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.14.174 x86_64)

   * Documentation:  https://help.ubuntu.com
   * Management:     https://landscape.canonical.com
   * Support:        https://ubuntu.com/advantage
  This system has been minimized by removing packages and content that are
  not required on a system that users do not log into.

  To restore this content, you can run the 'unminimize' command.
  root@ubuntu-fc-uvm:~#
  ```

    





