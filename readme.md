# NDv5 MI300X Quick Start Guide

Currently, we dont have an official, external image for ND v5 MI300X.  The following instructions detail how to start a VM and configure the OS for development.

1. Generate SSH key pair (optional):

        ssh-keygen -t rsa -b 2048 -f ~/.ssh/my-ssh-key

2. Start the VM:

        vmsku="Standard_ND96isr_MI300X_v5"
        image="Ubuntu2204"
        adminUser="MyUserName"
        rgName="MyResourceGroup"
        avsetName="MyAvSet"
        vmName="MyVM"
        keyPath="~/.ssh/my-ssh-key"
        location="canadacentral"

        az vm availability-set create --name $avsetName --resource-group $rgName  --platform-fault-domain-count 1  --platform-update-domain-count 1 --location $location
        
        az vm create --resource-group $rgName --name $vmName --image $image --admin-username $adminUser --size $vmsku --location $location --public-ip-sku Standard --disk-controller-type scsi --os-disk-size-gb 512 --availability-set $avsetName --security-type TrustedLaunch --enable-secure-boot false --ssh-key-values $keyPath.pub

3. SSH to the newly created VM:

        ssh -i $keyPath $adminUser@<public-ip>


4. Configure the OS.  Currently, this uses *BKC 24.05.03*.

        # kernel downgrade to 5.15.0-1059-azure
        sudo apt update
        sudo apt install -y aptitude
        sudo apt install linux-headers-5.15.0-1059-azure
        sudo aptitude install linux-image-5.15.0-1059-azure
        sudo apt purge linux-headers-6.5.0-1017-azure
        sudo apt purge linux-image-6.5.0-1017-azure
        sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.15.0-1059-azure"/' /etc/default/grub
        sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="panic=0 nowatchdog msr.allow_writes=on nokaslr amdgpu.noretry=1 pci=realloc=off console=ttyS0,115200n8 video=astdrmfb video=efifb:off ibt=off"/' /etc/default/grub
        sudo update-grub
        sudo reboot
        
        # device driver install 
        tar -xvzf  ROCm-6.1-13556-amdgpu1734616-BKC-ubuntu2204.tar
        cd ROCm-6.1-13556-amdgpu1734616-BKC-ubuntu2204/
        chmod u+x amdgpu-install
        ./amdgpu-install --usecase=rocm
        
        echo "blacklist amdgpu" |sudo tee -a /etc/modprobe.d/amdgpu.conf
        sudo update-initramfs -u -k all
        sudo usermod -a -G video $USER
        sudo usermod -a -G render $USER
        sudo reboot
         
        # load device driver
        sudo modprobe -r hyperv_drm
        sudo modprobe  amdgpu ip_block_mask=0x7f
        rocm-smi
