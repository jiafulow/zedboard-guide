# zedboard-guide

## Overview

- ZedBoard specs (Rev D):
  - Xilinx Zynq-7000 AP SoC XC7Z020-CLG484
  - Dual-core ARM Cortex A9
    - ARMv7-A architecture, r3p0 revision
  - Memory:
    - 512 MB DDR3 SD RAM
    - 256 Mb Quad-SPI flash memory
    - Static memory controller
    - 256 KB RAM in the on-chip memory
    - 4 GB SD card
  - Onboard USB-JTAG Programming
  - 10/100/1000 Ethernet
  - USB OTG 2.0 and USB-UART
  - PS & PL I/O expansion (FMC, Pmod™, XADC)
    - GPIO with four 32-bit banks, of which up to 54 bits can be used with the PS I/O and up to 64 bits connected to the PL
    - Up to 54 flexible multiplexed I/O (MIO) for peripheral pin assignments
  - Multiple displays (1080p HDMI, 8-bit VGA, 128 x 32 OLED)
  - I2S Audio CODEC
  - See http://www.digilentinc.com/zedboard

- ZedBoard images  
  (from http://zedboard.org/product/zedboard)

<table>
<tr>
<th>Front</th>
<th>Back</th>
</tr>
<tr>
<td><img src="/images/ZedBoard_RevA_sideA_0_0.jpg" width="240px" alt="Front"/></td>
<td><img src="/images/ZedBoard_RevA_sideB_0_0.jpg" width="240px" alt="Back"/></td>
</tr>
<tr>
<th>Functional Overlay</th>
<th>Block Diagram</th>
</tr>
<tr>
<td><img src="/images/Front-image-of-board_0.jpg" width="240px" alt="Functional Overlay"/></td>
<td><img src="/images/block_diagram_0_0.jpg" width="240px" alt="Block Diagram"/></td>
</tr>
</table>

- Zynq-7000 AP SoC overview  
  (from http://www.xilinx.com/support/documentation/user_guides/ug585-Zynq-7000-TRM.pdf)

<table>
<tr>
<td><img src="/images/ZynqBlockDiagram.png" width="480px" alt="Zynq-7000 AP SoC overview"/></td>
</tr>
</table>

- Host OS:
  - Ubuntu Linux 14.04.3 LTS (GNU/Linux 3.16.0-50-generic x86_64)
    - See http://releases.ubuntu.com/14.04/ 

- Xilinx applications:
  - Vivado Design Suite (including SDK) 2015.2
    - See http://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools/2015-2.html
  - PetaLinux 2015.2
    - See http://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools/2015-2.html

## Installation

- Install Xilinx applications:

```
# Vivado
tar -zxvf Xilinx_Vivado_SDK_Lin_2015.2_0626_1.tar.gz
cd Xilinx_Vivado_SDK_Lin_2015.2_0626_1
sudo ./xsetup

# PetaLinux
sudo ./petalinux-v2015.2.1-final-installer.run /opt/PetaLinux
```

- Install necessary packages from official Ubuntu repository:

```
sudo apt-get install gawk bison flex tftpd ncurses-dev zlib1g-dev libssl-dev
# The following are 32-bit libraries
sudo apt-get install lib32z1 lib32ncurses5 lib32bz2-1.0 lib32stdc++6
```

- Install cable drivers:

```
cd /opt/Xilinx/Vivado/2015.2/data/xicom/cable_drivers/lin64/install_script/install_drivers
sudo ./install_drivers 
```
- Change /bin/sh to bash

```
sudo dpkg-reconfigure dash
# Select <No>
```

- Install serial terminal console:

```
sudo apt-get install gtkterm
sudo addgroup `whoami` dialout 
sudo gtkterm
# Go to Configuration -> Port, select the following:
#   Port     : /dev/ttyACM0
#   Baud Rate: 115200
```

- Setup TFTP server
  - Create a text file /etc/xinetd.d/tftp with the following content:

  ```
  service tftp
  {
  protocol        = udp
  port            = 69
  socket_type     = dgram
  wait            = yes
  user            = nobody
  server          = /usr/sbin/in.tftpd
  server_args     = /tftpboot
  disable         = no
  }
  ```

  - Create a directory /tftpboot

  ```
  sudo mkdir /tftpboot
  sudo chmod -R 777 /tftpboot
  sudo chown -R nobody /tftpboot
  ```

  - Restart the xinetd service

  ```
  sudo service xinetd restart
  ```

- Setup DHCP server
  - Set a network connection profile with

   ```
   ip address: 192.168.1.1
   netmask   : 255.255.255.0
   gateway   : 192.168.1.100
   ```

  - Install DHCP server

   ```
   sudo apt-get install isc-dhcp-server
   ```

  - Edit /etc/dhcp/dhcpd.conf and /etc/default/isc-dhcp-server according to https://help.ubuntu.com/lts/serverguide/dhcp.html

  - Restart

    ```
    sudo service isc-dhcp-server restart
    ```

## Startup

- Get a license from http://www.xilinx.com/getlicense
  - Use the voucher to get a node-locked license (which will be emailed to you)
  - Alternatively, one can get the WebPACK license for free
  - In Vivado, select Help -> Manage License..., load the license Xilinx.lic
- Setup the environments:

```
source /opt/Xilinx/Vivado/2015.2/settings64.sh
source /opt/Xilinx/SDK/2015.2/settings64.sh
source /opt/PetaLinux/petalinux-v2015.2.1-final/settings.sh
```

## Useful info

- Zynq-7000 Technical Reference Manual (UG585)  
  http://www.xilinx.com/support/documentation/user_guides/ug585-Zynq-7000-TRM.pdf

- Zynq-7000 memory map  
  (from http://www.xilinx.com/support/documentation/data_sheets/ds190-Zynq-7000-Overview.pdf)

<table>
<tr>
<td><img src="/images/ZynqMemoryMap.png" width="480px" alt="Zynq-7000 memory map"/></td>
</tr>
</table>

- ZedBoard schematics  
  http://zedboard.org/sites/default/files/documentations/ZedBoard_RevD.2_Schematic_130516.pdf
  
- ZedBoard Out Of Box (OOB) SD card image and source can be downloaded from Digilent website: http://www.digilentinc.com/Data/Products/ZEDBOARD/ZedBoard_OOB_Design.zip
   - A backup copy of the OOB SD card can be found in the 'bootimage' directory

- PetaLinux Tools
  - See http://www.xilinx.com/tools/petalinux-sdk.htm
  - Note that PetaLinux v2015.2.1 is based on Yocto 1.8, which is based on Linux 3.19.
  - Note that the git trees for kernel and u-boot are
    - https://github.com/Xilinx/linux-xlnx
    - https://github.com/Xilinx/u-boot-xlnx

## Tutorials
1. ZedBoard Getting Started Guide  
   http://zedboard.org/sites/default/files/documentations/GS-AES-Z7EV-7Z020-G-V7.pdf

1. Avnet Zynq SpeedWay Workshops  
   http://zedboard.org/support/trainings-and-videos
   - Developing Zynq-7000 All Programmable SoC Hardware  
     http://zedboard.org/node/2563/
   - Developing Zynq-7000 All Programmable SoC Software  
     http://zedboard.org/node/2540/
   - PetaLinux for the Zynq-7000 All Programmable SoC  
     http://zedboard.org/course/petalinux-zynq%C2%AE-7000-all-programmable-soc

1. Zynq Design From Scratch blog by Sven Andersson  
   http://svenand.blogdrive.com/

1. Vivado Design Suite Tutorial: Embedded Processor Hardware Design (UG940)  
   http://www.xilinx.com/support/documentation/sw_manuals/xilinx2015_2/ug940-vivado-tutorial-embedded-design.pdf
   - Remember to download the tutorial design files

1. Xilinx University Program Workshops  
   http://www.xilinx.com/support/university/workshops.html
   - Embedded System Design Flow on Zynq  
     http://www.xilinx.com/support/university/vivado/vivado-workshops/Vivado-embedded-design-flow-zynq.html

## Known Issues

- Release Notes and Known Issues for PetaLinux 2013.04 and later (AR# 55776)
  - http://www.xilinx.com/support/answers/55776.html
