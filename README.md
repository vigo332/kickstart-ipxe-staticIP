# Using Kickstart to Remotely Upgrade RHEL 6 to RHEL 7 from HTTP server

## Background
My business unit manages about 100 workstations running RHEL 6 server edition in different networks and buildings. We need to upgrade the OS to RHEL 7 to match our HPC system. These workstations were added by disk imaging over time, but they have different disk sizes, partition layouts and hardware types. Although users of the WS use network filesystems, most of the local drives were used by users.

The upgrade requirements:

+ No loss of certain partitions containing user data. Ideally, preserve all the partition layouts. 
+ Very limited time window for the workstations, only 2 days. So the upgrade needs to be performed in parallel. Ideally, with careful preparasion, rebooting the RHEL6 servers will automatically install the RHEL7 OS from network.
+ Upgrade procedure needs to be generic to handle different hardware types include Dell Precision workstations with 1 TB HDD, Lenovo Thinkstation with 1 TB SSD, Intel NUC thin clients with 250 or 256 GB SSD, and Lenovo Think Tiny with 250 GB NVMe drives. 

Red Hat provides in-place major versions [upgrade tool](https://access.redhat.com/solutions/637583) from RHEL 6 to 7 for only server editions except x86. However, for mass upgrade of a pool of workstation servers, we found though this tool definitely works, it is a bit troublesome dealing with yum packages. It also limits to BIOS boot, no EFI boot loader supported. Servers with UEFI boot needs to convert to BIOS boot, which needs to deal with grub and GPT. Since our servers were installed with UEFI boot, changing the booting method and boot partition is tedious and error prone. We don't even want to physical touch the servers, everything must be done remotely. To circumvent these problems, we go with kickstart installation with [iPXE](http://ipxe.org/) over the network because this allows lots of customizations.


## Solution in High Level

Hardware requirement: the BIOS of the machines need to support UEFI boot and hence PXE/iPXE. This is normally supported by commodity hardwares which are not very old. 

+ The RHEL6 system will first boot into a customized iPXE binary. This could be done by placing the iPXE binary, eg. ipxe.efi in the /boot/efi/EFI/redhat directory (Note /boot and /boot/efi are mounted from different partitions, say hd0,1 and hd0,0, respectively if there is only one hard drive). The default boot sequence of the /etc/efi/EFI/redhat/grub.conf will be changed to the iPXE entry so that rebooting will start the iPXE efi to chainload the network kernel. 

+ The ipxe.efi binary needs to be prebuilt with an EMBED script setting the network for the iPXE environment. 

+ The iPXE environment will then start the network, chain a customized ipxe script (according to the network/IP of the RHEL6) from the http/ftp server. 

+ The chained iPXE script will also set the network on active interface, specify the initrd, kernel image kickstart file, rootfs (squashfs.img) and inst.repo from the http server, and boot into network kernel to install the RHEL7. 

Note: Since we are installing the OS from network remotely, it is crucial to make sure that in every step the workstations have working network. The network is configured 4 times:
1. Building the customized ipxe.efi binary with embed script,
2. The chained script also sets the network,
3. The network needs to be specified as arguments for the RHEL7 kernel on http server,
4. The kickstart file needs to set the network to grab packages from repository on the http server.
All the IPs, MAC addresses could be obtained before the upgrade, and will be used to customize the ipxe and kickstart files by scripting.


## Step by step

Note: the template files used below can all be customized for mass clients using scripts. 

1. On a server (RHEL) accessed by the target workstations, install apache and point the root directory to the extracted RHEL 7.3 iso directory, like http://<server>/ftp/tree. Put the customized chainloading ipxe scripts in http://<server>/ftp/ipxe, and the kickstart files to http://<server>/ftp/ks-files/

2. Obtain the IPs, MAC addresses, netmasks, gateways and MTUs (if jumbo frame enabled) of the target servers. This can be done using shell scripts. 

3. Install and compile ipxe
   ```git clone http://git.ipxe.org/ipxe.git
      yum install mtools binutils xz-devel -y
      cd ipxe/src
      make

      # To make a UEFI supported ipxe image with a custom script (setting IP)
      make bin-x86_64-efi/ipxe.efi EMBED=my_embed_script.ipxe
   ```

4. Prepare the embed scripts. See my_embed_script.ipxe for template.
   Build/make the customized ipxe.efi.

   ```
      # To make a UEFI supported ipxe image with a custom script (setting IP)
      make bin-x86_64-efi/ipxe.efi EMBED=my_embed_script.ipxe
   ```

   Copy the ipxe.efi to /boot/efi/EFI/redhat/ipxe.efi of each target server.

5. Change the /etc/efi/EFI/redhat/grub.conf, add one section for iPXE binary so that when reboot, a "Use iPXE" tab will show in the grub menu. For example, I have 2 rhel 6 kernels in menu, set the default kernel to 2 will used the newly added third ipxe menu. 

    ```
    default=2
    title Use iPXE 
        root (hd0,0)
        chainloader /EFI/redhat/ipxe.efi
    ```

6. Prepare the chainloader ipxe script, see my_chained_script.ipxe for template.

7. Prepare the kickstart file for each workstation, see my_kickstart.ks.cfg. 

   Several notes:

   * The network in the kickstart.ks.cfg does not need to be set since it was set as the kernel cmdline arguments in the chained script. The ifcfg-ifname file will be written during kickstart by the kernel. To have persistent network after kickstart, bring this interface up in the post script. 

   * We disabled RHEL [consistent network device naming](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/ch-consistent_network_device_naming) using `biosdevname=0` and `net.ifnames=0`. In this case, the IP address of the target server needs always be binded to the MAC address, and this can be collected before kickstart. 

   * Also the partition table should also be queried before kickstart so that the kickstart.cfg file can use the existing partitions, volume groups and LVMs (lsblk -bl). Use the `--useexisting --noformat` flags to preserve the existing partitions and logvols (eg, home and /shared, / needs to be purged.). Since all the workstations have the same LVM names but just varying sizes, the numbers, vgname and logvol can be `sed`ed.



