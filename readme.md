# NDv5 MI300X Quick Start Guide

Currently, we dont have an official, external image for ND v5 MI300X.  The following instructions detail how to start a VM and configure the OS for development.

1. Start the VM:

        base="mi300x_vm"
        vmsku="Standard_ND96isr_MI300X_v5"
        image="Ubuntu2204"
        pathPub="/home/shaunpur/.ssh/id_rsa.pub"
        adminUser="shaunpur"
        rgName="pcue-cca-rg"
        avsetName="avset-mi300-$base"
        vmName="test-$base"
        location="canadacentral"
        az vm availability-set create --name $avsetName --resource-group $rgName  --platform-fault-domain-count 1  --platform-update-domain-count 1 --location $location
        az vm create --resource-group $rgName --name $vmName --image $image --admin-username $adminUser --size $vmsku --location $location --public-ip-sku Standard --disk-controller-type scsi --os-disk-size-gb 512 --availability-set $avsetName --security-type TrustedLaunch --enable-secure-boot false --ssh-key-values $pathPub

2. Configure the OS.  Currently, this uses *BKC 24.06.03*.

        # kernel downgrade to 5.15.0-1059-azure
        sudo apt update
        sudo apt install -y aptitude
        sudo apt install linux-headers-5.15.0-1059-azure
        sudo aptitude install linux-image-5.15.0-1059-azure
        sudo apt purge linux-headers-6.5.0-1017-azure
        sudo apt purge linux-image-6.5.0-1017-azure
        sudo vim /etc/default/grub #GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.15.0-1059-azure"
        sudo vim /etc/default/grub #GRUB_CMDLINE_LINUX="panic=0 nowatchdog msr.allow_writes=on nokaslr amdgpu.noretry=1 pci=realloc=off console=ttyS0,115200n8 video=astdrmfb video=efifb:off ibt=off"
        sudo update-grub
        sudo reboot
        
        # device driver install 
        tar -xvzf  ROCm-6.2-13611-BKC-ubuntu2204.tar.gz
        cd ROCm-6.2-13611-BKC-ubuntu2204/
        chmod u+x amdgpu-install
        ./amdgpu-install --usecase=rocm
        
        echo "blacklist amdgpu" |sudo tee -a /etc/modprobe.d/amdgpu.conf
        sudo update-initramfs -u -k all
        sudo usermod -a -G video $USER
        sudo usermod -a -G render $USER
        sudo reboot
         
        # load device driver
        sudo modprobe -r ast
        sudo modprobe -r hyperv_drm
        sudo modprobe  amdgpu ip_block_mask=0x7f
        rocm-smi
