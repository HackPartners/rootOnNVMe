# rootOnNVMe
Switch the rootfs to a NVMe SSD on the Jetson Xavier NX and Jetson AGX Xavier. Originaly taken from [this repo](https://github.com/HackPartners/rootOnNVMe)

These scripts install a service which runs at startup to point the rootfs to a SSD installed on /dev/nvme0 (the M.2 Key M slot).

This is taken from the NVIDIA Jetson AGX Xavier forum https://forums.developer.nvidia.com/t/how-to-boot-from-nvme-ssd/65147/22, written by user crazy_yorik (https://forums.developer.nvidia.com/u/crazy_yorick). Thank you crazy_yorik!

This procedure should be done on a fresh install of the SD card using JetPack 4.3+. Install the SSD into the M.2 Key M slot of the Jetson, and format it gpt, ext4, and setup a partition (p1). The AGX Xavier uses eMMC, the Xavier NX uses a SD card in the boot sequence.

## Instructions
Running through these steps will delete all data on the ssd, and also could risk complications with the node potentially requiring a reflash. **Backup any important data before you start**. Additionally, it is best to download and run through these instructions as a super user on the node, so please consider swapping to su and grabbing this repo in the su's home directory before starting.

* First, ensure that the ssd is connected into the node and available to use. Run the command `lsblk` and find the ssd you want to use. If it is not listed, double check connections. For the remaining steps, we will use the example of `/dev/sda` as the ssd in all commands. **Please ensure you change the ssd to the one in your node setup**

* This ssd now needs to be formatted as an ext4 drive. To do so, run `mkfs.ext4 $SSD_NAME` eg `mkfs.ext4 /dev/sda`. You can also use the disk utility GUI to do this.

* After this, we need to obtain the UUID of the ssd. Run the command `lsblk -o NAME,UUID` and make note of the UUID. 

* If you havent already, ensure you have swapped into super user from this point to make your life easier: `sudo su`

* We now need to create a default mounting point for the ssd, and to configure it to run on boot. First, create a new directory `/media/storage` and ensure it has the correct permissions:
```
mkdir -p /media/storage && chmod 777 /media/storage
```

* Now we edit the fstab file to configure the auto mounting point on boot. Edit `etc/fstab` with your text editor of choice and add the following line, including the UUID listed previously. The white spaces below are tabs, please keep this consistent:
```
UUID="Enter the UUID within these quotes"     /media/storage  auto    0       2
```

* To check the mount runs properly, run the command `mount -a`. Use `lsblk` to check the ssd's MOUNTPOINT. If this is not listed as `/media/storage`, or if there is an error message, please check the ssd UUID and fstab file formatting in the previous steps before continuing

* Once happy the mounting works, unmount the ssd using `umount $SSD_NAME` eg `umount /dev/sda`

* We now need to edit the scripts and config files in this repo to match our node's configuration. Followng on from above, the following examples use /dev/sda **Ensure you replace this with your SSD's name and double check that all lines are correct to avoid something going wrong**

copy-rootfs-ssd.sh:

```
Line 3: sudo mount /dev/nvme0n1p1 /mnt -> sudo mount /dev/sda /mnt
```

data/setssdroot.service:

```
Line 8: ConditionPathExists=/dev/nvme0n1p1 -> ConditionPathExists=/dev/sda
```

data/setssdroot.sh:

```
Line 3: NVME_DRIVE="/dev/nvme0n1p1" -> NVME_DRIVE="/dev/sda"
```


* Now we can run the script to copy the rootfs of the eMMC/SD card to the SSD
```
$ ./copy-rootfs-ssd.sh
```

After running this command the ssd will be mounted on `/mnt`. **Check that files have been copied over to it before proceding**. It will be much harder to debug and check things are configured after the next step, and if something is incorrect you may need to reflash the node and start from scratch.

* Once happy, setup the service. This will copy the .service file to the correct location, and install a startup script to set the rootfs to the SSD.
```
$ ./setup-service.sh
```

* After setting up the service, reboot for the changes to take effect.

### Boot Notes
These script changes the rootfs to the SSD after the kernel image is loaded from the eMMC/SD card. For the Xavier NX, you will still need to have the SD card installed for booting. As of this writing, the default configuration of the Jetson NX does not allow direct booting from the NVMe.

### Upgrading
Once this service is installed, the rootfs will be on the SSD. If you upgrade to a newer version of L4T using OTA updates (using the NVIDIA .deb repository), you will need to also apply those changes to the SD card that you are booting from.

Typically this involves copying the /boot* directory and /lib/modules/\<kernel name\>/ from the SSD to the SD card. If they are different, then modules load will be 'tainted', that is, the modules version will not match the kernel version.


## Notes
* Initial Release, May 2020
* JetPack 4.4 DP
* Tested on Jetson Xavier NX

