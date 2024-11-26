# NDv5 MI300X Quick Start Guide

Currently, we dont have an official, external image for ND v5 MI300X.  The following instructions detail how to start a VM and configure the OS for development.

1. Generate SSH key pair (optional):

        ssh-keygen -t rsa -b 2048 -f ~/.ssh/my-ssh-key

2. Start the VM:

        vmsku="Standard_ND96isr_MI300X_v5"
        image="Ubuntu2204"
        adminUser="MyUserName"
        rgName="MyResourceGroup"
        vmName="MyVM"
        keyPath="~/.ssh/my-ssh-key"
        location="canadacentral"

        az group create --name $rgName --location $location
        
        az vm create --resource-group $rgName --name $vmName --image $image --admin-username $adminUser --size $vmsku --location $location --public-ip-sku Standard --disk-controller-type scsi --os-disk-size-gb 512  --security-type TrustedLaunch --enable-secure-boot false --ssh-key-values $keyPath.pub

3. SSH to the newly created VM:

        ssh -i $keyPath $adminUser@<public-ip>


4. Configure the OS.

        # kernel downgrade to 5.15.0-XXXX-azure kernel
        sudo apt install -y linux-headers-5.15.0-1073-azure
        sudo apt install -y linux-image-5.15.0-1073-azure
        # upgrade the default grub menu option.  Note: the HASH is tied to the kernel revision count, i.e. XXXX in 5.15.0-XXXX-azure.  It can be found in /boot/grub/grub.cfg
        sudo sed -i "s|GRUB_DEFAULT=.*|GRUB_DEFAULT='gnulinux-advanced-0b58668a-ba2e-4a00-b89a-3354b7a547d4>gnulinux-5.15.0-1073-azure-advanced-0b58668a-ba2e-4a00-b89a-3354b7a547d4'|g" /etc/default/grub
        sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="panic=0 nowatchdog msr.allow_writes=on nokaslr amdgpu.noretry=1 pci=realloc=off console=ttyS0,115200n8 video=astdrmfb video=efifb:off ibt=off"/' /etc/default/grub
        sudo update-grub
        sudo reboot
        sudo apt purge -y linux-headers-6.5.0-1025-azure linux-image-6.5.0-1025-azure
        
        # device driver install: https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/amdgpu-install.html
        wget https://repo.radeon.com/amdgpu-install/6.2.4/ubuntu/jammy/amdgpu-install_6.2.60204-1_all.deb
        sudo apt install ./amdgpu-install_6.2.60204-1_all.deb
        amdgpu-install --usecase=rocm
        
        echo "blacklist amdgpu" |sudo tee -a /etc/modprobe.d/amdgpu.conf
        sudo update-initramfs -u -k all
        sudo usermod -a -G video $USER
        sudo usermod -a -G render $USER
        sudo reboot
         
        # load device driver
        sudo modprobe -r hyperv_drm
        sudo modprobe  amdgpu ip_block_mask=0x7f
        rocm-smi
