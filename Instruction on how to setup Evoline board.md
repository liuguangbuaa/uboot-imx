# Instruction on how to setup Evoline board

## Equipment requirements

- Evoline main board
- Connector board
- Power supply
- Debugging tool
- Router

## Equipment setup

- Power supply set to 18V and 2A output

- Connect equipments as following picture:

  ![image-20220308154924027](C:\Users\320133902\AppData\Roaming\Typora\typora-user-images\image-20220308154924027.png)

- Download tftpd-hpa in ubuntu by `sudo apt-get install tftpd-hpa`

  open file `sudo vim /etc/default/tftpd-hpa` to check your default TFTP_DIRECTORY

  modify the permission of TFTP_DIRECTORY to 777 by `chmod 777 your_TFTP_DIRECTORY`

- Save your U-BOOT and Yocto images in the TFTP_DIRECTORY, here taking `u-boot-dtb-mx6evomon_dev.imx`  and `iv.wic`as an example: (`/srv/tftp` is my TFTP_DIRECTORY)

  ![image-20220302151743964](C:\Users\320133902\AppData\Roaming\Typora\typora-user-images\image-20220302151743964.png)

- Check your VM IP by typing `ifconfig` in terminal, taking it as the `serverip` (`192.168.72.131` is my `serverip`)

  ![image-20220302152435391](C:\Users\320133902\AppData\Roaming\Typora\typora-user-images\image-20220302152435391.png)

- Download `PUTTY` and setup as following:

  <img src="C:\Users\320133902\AppData\Roaming\Typora\typora-user-images\image-20220302153037666.png" alt="image-20220302153037666" style="zoom: 67%;" />

  where the Serial line `COM3` is the PC USB port which connected to the debugging tool `console to computer`  and it could be different from each other's PC.

  

- Turn on the power supply and open the `PUTTY` and you should see rolling information in `PUTTY` :

  <img src="C:\Users\320133902\AppData\Roaming\Typora\typora-user-images\image-20220302153753850.png" alt="image-20220302153753850" style="zoom: 67%;" />

- Once the rolling is stop and showing `restart at 3 seconds`, you should key `osum` to enter bootloader command-line **AS SOON AS POSSIBLE**. Once enter the command-line, there will be a time limit for operation, if you meet the time limit, the system will be reboot.

  

## Images Flashing

- Enter the command line of EVOLINE board, keying `usb start`

- Set up the environment by using 

  `setenv ipaddr 192.168.1.102`
  `setenv gatewayip 192.168.1.1`
  `setenv serverip 192.168.72.131`

  the `ipaddr` is the ip address for your main board and it can be modify by yourself `192.168.1.xxx`, here `102` is for example.

  `serverip` is the ip for your VM as mentioned above.

- Key `saveenv` to save environment and `printenv` can check your environment setting

- To flash your UBOOT image, keying `tftp 192.168.72.131:u-boot-dtb-mx6evomon_dev.imx` 

  the UBOOT image in your TFTP_DIRECTORY will be transmitted to the main board.

  Then, keying `sf probe 2:0; sf erase 0 70000; sf write ${loadaddr} 400 70000` 

- To flash Yocto, keying `tftp 192.168.72.131:iv.wic`
  `mmc write ${loadaddr} 0 0x457fe` 

  `0x457fe` is the size of your yocto image divided to 512 and changed to hex, and it should be dividable to 512.

  There should be some information showing that flashing is done.

- If flash successfully, restart you board, the UBOOT should be able to jump to linux kernal, if not, please keying 

  `setenv bootcmd 'run basicargs; run mmcargs; ext2load mmc 0:1 ${loadaddr} zImage; ext2load mmc 0:1 ${fdt_addr} imx6dp-calimon.dtb; bootz ${loadaddr} - ${fdt_addr}'
  setenv mmcrootfstype ext4 rootwait
  saveenv` 

- `printenv` to check the following variables are correct:

  - mmcargs=setenv bootargs ${bootargs} root=${mmcroot} rootfstype=${mmcrootfstype}
  - mmcroot=/dev/mmcblk0p2 ro
  - mmcrootfstype=ext4 rootwait
  - bootcmd=run basicargs; run mmcargs; ext2load mmc 0:1 ${loadaddr} zImage; ext2load mmc 0:1 ${fdt_addr} imx6dp-calimon.dtb; bootz ${loadaddr} - ${fdt_addr}

  if not the same, can use `setenv` to setup.

- `saveenv` to save all once modify.

## Linux system setup in board

- Once all images flashed successfully, it will jump to linux kernel, and when it stops, you need to key in `root`

  ![image-20220302171719680](C:\Users\320133902\AppData\Roaming\Typora\typora-user-images\image-20220302171719680.png)

  then you enter the linux system on board.

- Keying `ip a` to check all available netcard, select your netcard and key `cd /etc/systemd/network` , then `vi 10-wired-static.network` and input the content as follow:

  `[Match] `

  `Name= the available netcard name you choose `
  
  `[Network] `
  
  `DHCP=ipv4`
  
  
  
  if system output `Read file system only`, key `mount -o remount,rw /` to overwrite the system.
  
  Keying `systemctl restart systemd-networkd` and `systemctl enable systemd-networkd` to restart and enable your setup.
  
- Generate ssh keys with command:

  `cd /etc/ssh`

  `ssh-keygen -t rsa -f ssh_host_rsa_key`

  `ssh-keygen -t ecdsa -f ssh_host_ecdsa_key`

  `ssh-keygen -t ed25519 -f ssh_host_ed25519_key`

  

- `vi /etc/ssh/sshd_config_readonly`  and modify the following content:

  ![image-20220303091206150](C:\Users\320133902\AppData\Roaming\Typora\typora-user-images\image-20220303091206150.png)
  
- `cd /etc/systemd/system/multi-user.target.wants ` 

  `ls -l systemd-networkd.service`

  save the output path `../../../../lib/systemd/system`

- Create two soft connections by

  `cd /ect/systemd/system/getty.target.wants `

  `ln -s ../../../../lib/systemd/system/systemd-networkd.service systemd-networkd.service`

  ``ln -s ../../../../lib/systemd/system/sshd.socket sshd.socket``

- `reboot` the system

- `ifconfig` to check the ip address of the board

- Open windows powershell in your windows system, input `ssh root@the ip address` to connect to the board.
