# Tracing IP with QEMU

## Download QEMU (V10.0.3)
````sh
wget https://github.com/qemu/qemu/archive/refs/tags/v10.0.3.zip
unzip v10.0.3.zip
````

## Download dependencies
````sh
# QEMU Build dependencies
sudo apt install ninja-build python3-sphinx-rtd-theme python3-sphinx libglib2.0-dev libncurses-dev flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev libcapstone-dev python3-venv

# Install the official qemu build-dep for good measure
sudo apt build-dep qemu


sudo apt install gcc-aarch64-linux-gnu #  ARM gcc for comiling arm kernel
````

## Build QEMU
````sh
cd qemu-10.0.3

# Download the modified cpu-exec.c file
rm -f accel/tcg/cpu-exec.c
wget https://gist.githubusercontent.com/ubdussamad/fa55885f57397219171a8f30332408be/raw/409365c43e8dc85350573406a476b933c92154d6/10.0.3-cpu-exec-mod.c accel/tcg/cpu-exec.c

rm -rf accel/tcg/trace-events
wget https://gist.githubusercontent.com/ubdussamad/0bc723df0a88c72a959a25ce03033448/raw/c89681443540beebd74cc8cc8412a2ca5f440d90/10.0.3-trace-events-mod accel/tcg/trace-events

mkdir build
cd build

# Configure QEMU
#
# Details about config options:
#  - Selected AArch64 target
#  - Enable log trace backend for tracing
../configure --target-list=aarch64-softmmu --enable-trace-backends=log
make -j$(nproc)  # Build QEMU

cd ../ # Change to the parent directory of qemu-10.0.3
````


## Build Simple rootfs and Kernel with buildroot
```sh

# Download and extract buildroot in the current directory
wget https://buildroot.org/downloads/buildroot-2025.02.tar.xz
tar -xvf buildroot-2025.02.tar.xz
cd buildroot-2025.02
make qemu_aarch64_virt_defconfig      # good starting point for QEMU/virt board
make menuconfig

make -j$(nproc)            # Build the rootfs and Kernel (this downloads the kernel locally so we can edit the configs)

# Clean the kernel as we need a fresh copy of the kernel with debug symbols
make linux-dirclean
rm -rf output/images/Image output/images/vmlinux  # Clean the output images directory

# Then enable debug symbols by:
make linux-menuconfig
# In the menuconfig, navigate to:
#   Kernel hacking -> Kernel debugging (Enable it)
# Double check the config file in `output/build/linux-6.*/.config` to ensure the following options are set:
#   CONFIG_DEBUG_INFO_NONE=n
#   CONFIG_DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT=y
#   CONFIG_DEBUG_INFO=y
#   CONFIG_AS_HAS_NON_CONST_ULEB128=y
#   CONFIG_DEBUG_INFO_NONE is not set
#   CONFIG_DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT=y

make linux-rebuild -j$(nproc)            # Rebuild the kernel with debug symbols

# Now you'll have to copy the vmlinux (bin with debug symbols) file to the output/images directory
cp output/build/linux-6.*/vmlinux output/images/vmlinux

# Now you have everything set up to run and collect traces

cd ../ # Get back to the parent directory of buildroot-2025.02

```

Once you're done building the kernel and rootfs through buildroot, you'll see the outputs in
`buildroot-2025.02/output/images` ,  you should see `Image, vmlinux, rootfs.ext2` files there.

## Run QEMU with the built kernel and rootfs

Finally, run qemu with the newly build image by:
```sh


# Env variables for the rootfs and kernel images
BR_FS_IMG=$(pwd)/buildroot-2025.02/output/images/rootfs.ext4
BR_KERNEL_IMG=$(pwd)/buildroot-2025.02/output/images/Image
BR_KERNEL_VMLINUX=$(pwd)/buildroot-2025.02/output/images/vmlinux

# Sanity Check: Ensure everything exists by looping through the files
for file in ${BR_FS_IMG} ${BR_KERNEL_IMG} ${BR_KERNEL_VMLINUX}; do
    if [ ! -f "$file" ]; then
        echo "Error: File $file does not exist."
        exit 1
    else 
        echo "Found $file"
    fi
done

# Run QEMU with the built kernel and rootfs
# Note: The qemu machine's output will be saved in trace-stdout.tmp, to interact with the machine, open the file in a text editor and then type your commands 
# (e.g. root) in the terminal that is running QEMU to interact with the machine. (I know its a hack but a proper solution is on the way)
qemu-10.0.3/build/qemu-system-aarch64 -trace exec_tb_block -M virt -cpu cortex-a53 -m 6144 -nographic -smp 4 -kernel ${BR_KERNEL_IMG} -append "rootwait nokaslr root=/dev/vda console=ttyAMA0" -drive file=${BR_FS_IMG},if=none,format=raw,id=hd0 -device virtio-blk-device,drive=hd0  -nographic 3>&1 1>trace-stdout.tmp 2>&3 | awk '!seen[$0]++' > _ip.log

# Operate the machine by logging in as root and then doing whatever you want to do, then poweroff the machine by typing `poweroff` in the terminal running QEMU.

rm trace-stdout.tmp # If you want to remove the temporary file after extracting the IP addresses

# Filter the IP address from the trace output
awk '{ if (match($0, /0xffff[0-9a-fA-F]+/)) print substr($0, RSTART, RLENGTH) }' _ip.log  > ip.log && rm _ip.log


# Run addr2line to get the function names from the vmlinux file
cat ip.log | addr2line -e ${BR_KERNEL_VMLINUX} > src_lines.log
