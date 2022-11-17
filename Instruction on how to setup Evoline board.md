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




autoload=no
basicargs=setenv bootargs mtdoops.mtddev=oopslog mtdoops.record_size=16384 vmalloc=392MB consoleblank=0 quiet console=ttymxc0,${baudrate} bootRev=IntelliVue-OSS-R18-181 ethaddr=${usbethaddr} user_debug=30
baudrate=115200
board_id=1296
bootargs_systemd=if test ${card_id} = 11a0; then setenv systemd_target iv-calimon.target; else setenv systemd_target iv-evomon.target; fi; setenv bootargs ${bootargs} systemd.unit=${systemd_target}
bootcmd=if test ${make_upgrade} -eq 0 ; then echo Booting Normal ; run mmcboot ; else ; echo Entering Upgrade ; if test ${do_buzzer} -eq 1 ; then delphi_buzzer on ; fi ; run upgrade ; fi ; reset
bootcount=1
bootdelay=3
bootfile=Delphi.uImage
bootp_vci=8a1e93f5-107d-4ac9-9c54-63d12c75e1ca
card_id=11a0
card_rev=0004
dhcp_srv_vendor-class-identifier=afb0ec93-0d32-4c9a-81e3-ea4b4a20932f
do_buzzer=0
dtb=11a0-0004.dtb
eth1addr=00:09:fb:05:c3:d2
ethaddr=00:09:fb:05:c3:d3
ethprime=sms950
fdt_addr=0x18000000
fdtcontroladdr=8fd5ee18
loadaddr=0x12000000
mmc_detect_bootpart=true
mmc_load_fit_image=ext2load mmc ${mmcdev}:${mmcpart} ${loadaddr} boot/uImage C00000; if test $? != 0; then reset ; fi;
mmcargs=setenv bootargs ${bootargs} root=${mmcroot} rootfstype=${mmcrootfstype}
mmcboot=run mmc_load_fit_image; run basicargs; run mmcargs; run bootargs_systemd; run startuimage;
mmcdev=0
mmcpart=1
mmcroot=/dev/mmcblk0p1 ro
mmcrootfstype=ext3 rootwait
mtdparts=mtdparts=m25p32:512k(u-boot),64k(u-boot-env1),64k(u-boot-env2),512k(activity);delphi-sram:64k(oopslog),512k(logfiles),-(buffmem)
netargs=setenv bootargs ${bootargs} root=/dev/nfs ip=dhcp nfsroot=${rootpath},v3,tcp
netboot=echo Booting from net ...; usb start; setenv ethact ${ethprime}; setenv scriptaddr 0x14000000; run basicargs; dhcp; tftp ${loadaddr} uImage; if test $? = 0; then tftp ${scriptaddr} npi-ramdisk.fit; else reset; fi; if test $? = 0; then if test ${card_id} = 11a0; then setenv netbootconfig imx6dp-calimon; else setenv netbootconfig imx6dp-evomon; fi; else reset; fi; if test $? = 0; then bootm ${loadaddr}:kernel ${scriptaddr}:ramdisk ${loadaddr}:${netbootconfig}; else reset; fi;
netretry=once
panel_card_id=7240
startuimage=fit_check_property_existence philips,card-id-rev-based-dtb; if test $? = 0; then fit_conf_find_compatible philips,card-id-rev,${card_id}-${card_rev}; if test $? != 0; then fit_conf_find_compatible philips,card-id,${card_id}; fi; if test -n ${fit_config} ; then bootm ${loadaddr}#${fit_config}; fi; fi;
stderr=serial
stdin=serial
stdout=serial
uimage=uImage
upgargs=setenv bootargs ${bootargs} ${mtdparts} supportip=${supportip} local_context=${local_context} ip=${ipaddr}::${gatewayip}:${netmask}:::
upgrade=usb start; setenv ethact ${ethprime}; run basicargs; if test ${local_context} != yes; then setenv netretry yes; dhcp; setenv supportip ${serverip}; tftp; else tftp ${supportip}:${bootfile}; fi; if test $? = 0; then run upgargs; usb stop; fit_remove_nodes_with_at_in_name; fit_check_property_existence philips,card-id-rev-based-dtb; if test $? = 0; then fit_conf_find_compatible philips,card-id-rev,${card_id}-${card_rev}; if test $? != 0; then fit_conf_find_compatible philips,card-id,${card_id}; fi; if test -n ${fit_config} ; then bootm ${loadaddr}#${fit_config}; fi; fi; fi; reset;
usbethaddr=00:09:fb:05:c3:d2

autoload=no
basicargs=setenv bootargs mtdoops.mtddev=oopslog mtdoops.record_size=16384 vmalloc=392MB consoleblank=0  console=ttymxc0,${baudrate} bootRev=IntelliVue-OSS-R18-26 ethaddr=${usbethaddr}
baudrate=115200
board_id=1296
bootargs_systemd=if test ${card_id} = 11a0; then setenv systemd_target iv-calimon.target; else setenv systemd_target iv-evomon.target; fi; setenv bootargs ${bootargs} systemd.unit=${systemd_target}
bootcmd=echo Manufacturing Build: Entering Upgrade only ; run upgrade
bootcount=1
bootdelay=3
bootfile=Delphi.uImage
bootp_vci=8a1e93f5-107d-4ac9-9c54-63d12c75e1ca
card_id=11a0
card_rev=0004
dhcp_srv_vendor-class-identifier=afb0ec93-0d32-4c9a-81e3-ea4b4a20932f
do_buzzer=0
dtb=11a0-0004.dtb
eth1addr=00:09:fb:05:c3:9e
ethaddr=00:09:fb:05:c3:9f
ethprime=sms950
fdt_addr=0x18000000
fdtcontroladdr=8fd5fe18
loadaddr=0x12000000
mmc_detect_bootpart=true
mmc_load_fit_image=ext2load mmc ${mmcdev}:${mmcpart} ${loadaddr} boot/uImage A00000; if test $? != 0; then reset ; fi;
mmcargs=setenv bootargs ${bootargs} root=${mmcroot} rootfstype=${mmcrootfstype}
mmcboot=run mmc_load_fit_image; run basicargs; run mmcargs; run bootargs_systemd; run startuimage;
mmcdev=0
mmcpart=1
mmcroot=/dev/mmcblk0p1 ro
mmcrootfstype=ext3 rootwait
mtdparts=mtdparts=m25p32:512k(u-boot),64k(u-boot-env1),64k(u-boot-env2),512k(activity);delphi-sram:64k(oopslog),512k(logfiles),-(buffmem)
netargs=setenv bootargs ${bootargs} root=/dev/nfs ip=dhcp nfsroot=${rootpath},v3,tcp
netboot=echo Booting from net ...; usb start; setenv ethact ${ethprime}; setenv scriptaddr 0x14000000; run basicargs; dhcp; tftp ${loadaddr} uImage; if test $? = 0; then tftp ${scriptaddr} npi-ramdisk.fit; else reset; fi; if test $? = 0; then if test ${card_id} = 11a0; then setenv netbootconfig imx6dp-calimon; else setenv netbootconfig imx6dp-evomon; fi; else reset; fi; if test $? = 0; then bootm ${loadaddr}:kernel ${scriptaddr}:ramdisk ${loadaddr}:${netbootconfig}; else reset; fi;
netretry=once
panel_card_id=7240
startuimage=fit_check_property_existence philips,card-id-rev-based-dtb; if test $? = 0; then fit_conf_find_compatible philips,card-id-rev,${card_id}-${card_rev}; if test $? != 0; then fit_conf_find_compatible philips,card-id,${card_id}; fi; if test -n ${fit_config} ; then bootm ${loadaddr}#${fit_config}; fi; fi;
stderr=serial
stdin=serial
stdout=serial
uimage=uImage
upgargs=setenv bootargs ${bootargs} ${mtdparts} supportip=${supportip} local_context=${local_context} ip=${ipaddr}::${gatewayip}:${netmask}:::
upgrade=usb start; setenv ethact ${ethprime}; run basicargs; if test ${local_context} != yes; then setenv netretry yes; dhcp; setenv supportip ${serverip}; tftp; else tftp ${supportip}:${bootfile}; fi; if test $? = 0; then run upgargs; usb stop; fit_check_property_existence philips,card-id-rev-based-dtb; if test $? = 0; then fit_conf_find_compatible philips,card-id-rev,${card_id}-${card_rev}; if test $? != 0; then fit_conf_find_compatible philips,card-id,${card_id}; fi; if test -n ${fit_config} ; then bootm ${loadaddr}#${fit_config}; fi; fi; fi; reset;
usbethaddr=00:09:fb:05:c3:9e
zboot=run mmc_detect_bootpart; ext2load mmc ${mmcdev}:${mmcpart} ${loadaddr} boot/zImage; ext2load mmc ${mmcdev}:${mmcpart} ${fdt_addr} boot/${card_id}.dtb; ext2load mmc ${mmcdev}:${mmcpart} ${fdt_addr} boot/${card_id}-${card_rev}.dtb; run basicargs; run mmcargs; run bootargs_systemd; bootz ${loadaddr} - ${fdt_addr};


