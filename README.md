# zedboard-guide

## Overview

ZedBoard specs (Rev D):
- Zynq-7000 AP SoC XC7Z020-CLG484
- Dual-core ARM Cortex A9
  - ARMv7-A architecture, r3p0 revision
- Memory:
  - 512 MB DDR3
  - 256 Mb Quad-SPI Flash
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

ZedBoard images 
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

Zynq-7000 AP SoC overview 
(from http://www.xilinx.com/support/documentation/user_guides/ug585-Zynq-7000-TRM.pdf)
<table>
<tr>
<td><img src="/images/ZynqBlockDiagram.png" width="480px" alt="Zynq-7000 AP SoC overview"/></td>
</tr>
</table>

Host OS:
- Ubuntu Linux 14.04.3 LTS (GNU/Linux 3.16.0-50-generic x86_64)

Xilinx applications:
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
sudo apt-get install gawk ncurses-dev zlib1g-dev libssl-dev
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


## Startup

- Get the licenses by launching Vivado. Select Help ->  Manage License...
  - The licenses should appear in the directory ~/.Xilinx
- Setup the environments:

```
source /opt/Xilinx/Vivado/2015.2/settings64.sh
source /opt/Xilinx/SDK/2015.2/settings64.sh
source /opt/PetaLinux/petalinux-v2015.2.1-final/settings.sh
```

