# alpine2firecracker
Converts an alpine container into an EXT4 filesystem for Firecracker.

## Usage
Run a container using your desired image and do any desired changes such as adding users, configuring networking, and installing applications. Then execute the script alpine2firecracker. This should generate a ext4 filesystem called rootfs.ext4 and a build log file called build.log
