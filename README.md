# alpine2firecracker
Converts an alpine container into an EXT4 filesystem for Firecracker.

## Usage
Run a container using your desired image and do any desired changes such as adding users, configuring networking, and installing applications. Then execute the script alpine2firecracker. This should generate a ext4 filesystem called rootfs.ext4 and a build log file called build.log

## Example
This example uses the container alpine-firecracker-example1 which shoukd exist on dockerhub's repo. The source for this container is in the directory of this repo under alpine-firecracker-example1. We will mount the working directory to /mnt of the container so that we can copy out the ext4 filesystem. This container is built to use DHCP for networking and expects a single network interface.

The following commands should result in the creation of a file called rootfs.ext4 in your working directory,
```
docker run --privileged -v $PWD:/mnt -it jodydadescott/alpine-firecracker-example1 sh
wget https://raw.githubusercontent.com/jodydadescott/alpine2firecracker/master/alpine2firecracker
sh alpine2firecracker
mv rootfs.ext4 /mnt
```

Now you are ready to run the image. The instructions below are based on the official repo [firecracker-microvm](https://github.com/firecracker-microvm/firecracker/blob/master/docs/getting-started.md). The following assumptions are made:
* Firecracker is installed
* A Linux bridge is configured for networking and named br0
* A DHCP server is configured on the br0 subnet
* You have a kernel which can be downloaded [here](https://github.com/jodydadescott/firecracker-kernel/releases/download/v4.16.0/vmlinuz-4.16)

You will need to open two terminal windows. On one window run the commands
```
rm -rf /tmp/example1.socket
firecracker --api-sock /tmp/example1.socket
```

Run this command to set the linux kernel image.
```
curl --unix-socket /tmp/example1.socket \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -i -X PUT http://localhost/boot-source \
  -d '{ "kernel_image_path": "./vmlinuz", "boot_args": "console=ttyS0 reboot=k panic=1 pci=off" }'
```
Run this command to add a single interface.
```
curl --unix-socket /tmp/example1.socket \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -i -X PUT http://localhost/network-interfaces/1 \
  -d '{ "iface_id": "1", "host_dev_name": "example1" }'
```
Run this command to bind the interface we just created to the linux bridge br0.
```
brctl addif br0 example1 && ifconfig example1 up
```
Run this coomand to set the roofs (the file we just created)
```
curl --unix-socket /tmp/example1.socket 
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -i -X PUT http://localhost/drives/rootfs \
  -d '{ "drive_id": "rootfs", "path_on_host": "./rootfs.ext4", "is_root_device": true, "is_read_only": false }'
```
Run this command to set the memory.
```
curl --unix-socket /tmp/example1.socket \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -i -X PUT http://localhost/machine-config \
  -d '{ "vcpu_count":1, "mem_size_mib":1024 }'
```
Finally run this command to boot the system.
```
curl --unix-socket /tmp/example1.socket \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -i -X PUT http://localhost/actions \
  -d '{ "action_type": "InstanceStart" }'
```


