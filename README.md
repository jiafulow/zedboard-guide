# zedboard-guide

## Overview

- ZedBoard specs (Rev D):
  - Xilinx Zynq-7000 AP SoC XC7Z020-CLG484-1
  - Dual-core ARM Cortex A9
    - ARMv7-A architecture, r3p0 revision
    - 667 MHz max clock frequency (speed grade -1)
    - L1 cache: 32 KB instruction, 32 KB data per processor
    - L2 cache: 512 KB
    - 256KB on chip memory (OCM)
  - Memory:
    - 512 MB DDR3 SD RAM
    - 256 Mb QSPI flash memory
    - Static memory controller
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
    - e.g.

  ```
  # Edit /etc/dhcp/dhcpd.conf
  subnet 192.168.1.0 netmask 255.255.255.0 {
   range 192.168.1.150 192.168.1.200;
   option routers 192.168.1.254;
   option domain-name-servers 192.168.1.1, 192.168.1.2;
   option domain-name "mydomain.example";
  }
  ```

  ```
  # Edit /etc/default/isc-dhcp-server
  INTERFACES="eth0"
  ```

  - Restart

  ```
  sudo service isc-dhcp-server restart
  ```

- Setup NFS server

  - Install NFS server

  ```
  sudo apt-get install nfs-kernel-server
  ```

  - Edit /etc/exports according to https://help.ubuntu.com/community/SettingUpNFSHowTo#Install_NFS_Server
    - e.g.

  ```
  # Edit /etc/exports
  <directory> 192.168.1.*(rw,sync,no_subtree_check)
  ```

  - Restart

  ```
  sudo service nfs-kernel-server restart
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

- References
  - ZedBoard Hardware User's Guide  
    http://zedboard.org/sites/default/files/documentations/ZedBoard_HW_UG_v2_2.pdf
  - ZedBoard Master Constraints  
    http://zedboard.org/sites/default/files/documentations/zedboard_master_XDC_RevC_D_v2.zip
  - Zynq-7000 Technical Reference Manual (UG585)  
    http://www.xilinx.com/support/documentation/user_guides/ug585-Zynq-7000-TRM.pdf
  - Zynq-7000 All Programmable SoC Overview (DS190)  
    http://www.xilinx.com/support/documentation/data_sheets/ds190-Zynq-7000-Overview.pdf
  - ARM Cortex-A9 Technical Reference Manual  
    http://infocenter.arm.com/help/topic/com.arm.doc.ddi0388g/DDI0388G_cortex_a9_r3p0_trm.pdf

- Zynq-7000 IP (ZYNQ7 Processing System) in Vivado 2015.2 with ZedBoard preset

<table>
<tr>
<td><img src="/images/ZynqIP.png" width="480px" alt="ZYNQ7 Processing System IP"/></td>
</tr>
</table>

- Zynq-7000 memory map  
  (from http://www.xilinx.com/support/documentation/data_sheets/ds190-Zynq-7000-Overview.pdf)

<table>
<tr>
<td><img src="/images/ZynqMemoryMap.png" width="480px" alt="Zynq-7000 memory map"/></td>
</tr>
</table>

- Zynq-7000 clocks
  - Generated by three programmable PLLs: ARM PLL, DDR PLL, I/O PLL
  - (Default) Input frequency: 33.333333 MHz, CPU clock ratio 6:2:1

- Zynq-7000 PS-PL interface
  - 2x 32-bit AXI general-purpose (GP) master interfaces
  - 2x 32-bit AXI general-purpose (GP) slave interfaces
  - 4x 32 or 64-bit AXI high-performance (HP) slave interfaces
  - 1x 64-bit AXI Accelerator Coherency Port (ACP) slave interface
  - 4x PS clock outputs to PL
  - 4x PS reset outputs to PL
  - 16x interrupts
  - DMA, event signals, EMIO, ...

- ZedBoard schematics  
  http://zedboard.org/sites/default/files/documentations/ZedBoard_RevD.2_Schematic_130516.pdf
  
- ZedBoard Out Of Box (OOB) SD card image and source can be downloaded from Digilent website: http://www.digilentinc.com/Data/Products/ZEDBOARD/ZedBoard_OOB_Design.zip
   - A backup copy of the OOB SD card can be found in the 'bootimage' directory

- PetaLinux Tools
  - See http://www.xilinx.com/tools/petalinux-sdk.htm
  - Also see http://www.wiki.xilinx.com/PetaLinux
  - PetaLinux v2015.2.1 is based on Yocto 1.8, which is based on Linux 3.19
  - PetaLinux includes BusyBox 1.23.1 and Dropbear SSH server
  - The git trees for PetaLinux kernel and u-boot are
    - https://github.com/Xilinx/linux-xlnx
    - https://github.com/Xilinx/u-boot-xlnx
  - Xilinx Wiki for Linux tools
    - http://www.wiki.xilinx.com/Getting+Started
    - http://www.wiki.xilinx.com/Linux+Drivers

- PetaLinux Workflow
  - `cd <PetaLinux_Project>`
  - `petalinux-create -t project -n software --template zynq`
  - `cd <Vivado_Export_to_SDK_Directory>`
  - `petalinux-config --get-hw-description -p <PetaLinux_Project>/software/`
    - Make sure that "primary sd" is selected in 
      - Subsystem AUTO Hardware Settings > Advanced bootable images storage Settings > boot image settings > image storage media
      - Subsystem AUTO Hardware Settings > Advanced bootable images storage Settings > kernel image settings > image storage media
    - (Optional) `petalinux-config -c rootfs`
    - (Optional) `petalinux-config -c kernel`
  - `petalinux-create -t apps -n myapp --template c --enable`
  - `petalinux-build`
    - Make necessary changes to device tree settings found in subsystems/linux/configs/device-tree/
  - `cd images/linux`
  - `petalinux-package --boot --fsbl zynq_fsbl.elf --fpga system_wrapper.bit --uboot`
    - Copy BOOT.BIN and image.ub (roughly 11 MB) to the SD card.
      - The SD card has to be formatted as FAT32.
    - Boot the ZedBoard with the SD card (make sure the jumpers are set correctly).

- PetaLinux netboot using TFTP
  - Use SD card for initial boot. Connect the ethernet cable. 
  - When the message "Hit any key to stop autoboot" shows, stop the autoboot. 
  - If an IP address was not obtained, run `dhcp`.
  - Run `set serverip 192.168.1.1`  (TFTP server IP).
  - Run `run netboot`.

- Acronym

<table style="width:480px">
<tr>
<th width="25%">Acronym</th>
<th width="75%">Definition</th>
</tr>

<tr><td>ACP         </td><td>Accelerator Coherency Port                 </td></tr>
<tr><td>AP SoC      </td><td>All Programmable System on a Chip          </td></tr>
<tr><td>APU         </td><td>Application Processor Unit                 </td></tr>
<tr><td>ASIC        </td><td>Application-Specific Integrated Circuit    </td></tr>
<tr><td>AXI         </td><td>Advanced eXtensible Interface              </td></tr>
<tr><td>BSP         </td><td>Board Support Package                      </td></tr>
<tr><td>DMA         </td><td>Direct Memory Access                       </td></tr>
<tr><td>DRM         </td><td>Direct Rendering Manager                   </td></tr>
<tr><td>DSP         </td><td>Digital Signal Processor                   </td></tr>
<tr><td>DTB         </td><td>Device Tree Binary                         </td></tr>
<tr><td>DTS         </td><td>Device Tree Source                         </td></tr>
<tr><td>EMIO        </td><td>Extended MIO                               </td></tr>
<tr><td>FIFO        </td><td>First In, First Out                        </td></tr>
<tr><td>FMC         </td><td>FPGA Mezzanine Card                        </td></tr>
<tr><td>FPGA        </td><td>Field-Programmable Gate Array              </td></tr>
<tr><td>FSBL        </td><td>First-Stage Boot Loader                    </td></tr>
<tr><td>GDB         </td><td>GNU Project Debugger                       </td></tr>
<tr><td>GIC         </td><td>Generic Interrupt Controller               </td></tr>
<tr><td>GNU         </td><td>GNU's Not Unix                             </td></tr>
<tr><td>GPIO        </td><td>General-Purpose I/O                        </td></tr>
<tr><td>HDL         </td><td>Hardware Description Language              </td></tr>
<tr><td>HLS         </td><td>High-Level Synthesis                       </td></tr>
<tr><td>I2C         </td><td>Inter-Integrated Circuit                   </td></tr>
<tr><td>IDE         </td><td>Integrated Development Environment         </td></tr>
<tr><td>IOP         </td><td>Input/Output Peripherals                   </td></tr>
<tr><td>IP          </td><td>Intellectual Property                      </td></tr>
<tr><td>IRQ         </td><td>Interrupt ReQuest                          </td></tr>
<tr><td>JTAG        </td><td>Joint Test Action Group                    </td></tr>
<tr><td>MIO         </td><td>Multiplexed I/O                            </td></tr>
<tr><td>MMU         </td><td>Memory Management Unit                     </td></tr>
<tr><td>OCM         </td><td>On-Chip Memory                             </td></tr>
<tr><td>OS          </td><td>Operating System                           </td></tr>
<tr><td>PL          </td><td>Programmable Logic (a.k.a. fabric)         </td></tr>
<tr><td>PLL         </td><td>Phase-Locked Loop                          </td></tr>
<tr><td>PS          </td><td>Processing System (a.k.a. processor)       </td></tr>
<tr><td>QSPI        </td><td>Quad Serial Peripheral Interface           </td></tr>
<tr><td>RTL         </td><td>Register Transfer Level                    </td></tr>
<tr><td>SCU         </td><td>Snoop Control Unit                         </td></tr>
<tr><td>SDK         </td><td>Software Development Kit                   </td></tr>
<tr><td>SIMD        </td><td>Single Instruction Multiple Data           </td></tr>
<tr><td>SoC         </td><td>System on a Chip                           </td></tr>
<tr><td>TLA         </td><td>Three-Letter Acronym                       </td></tr>
<tr><td>TTC         </td><td>Triple-Timer Counter                       </td></tr>
<tr><td>UART        </td><td>Universal Asynchronous Receiver/Transmitter</td></tr>
<tr><td>UIO         </td><td>Userspace I/O                              </td></tr>

</table>

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
   - Advanced Embedded System Design on Zynq  
     http://www.xilinx.com/support/university/vivado/vivado-workshops/Vivado-adv-embedded-design-zynq.html
   - Embedded Linux on Zynq  
     http://www.xilinx.com/support/university/vivado/vivado-workshops/Vivado-embedded-linux-zynq.html

1. Zynq-7000 All Programmable SoC: Embedded Design Tutorial (UG1165)  
   http://www.xilinx.com/support/documentation/sw_manuals/xilinx2015_2/ug1165-zynq-embedded-design-tutorial.pdf
   - Remember to download the tutorial design files

1. Zynq Base Targeted Reference Design (TRD) 2015.2  
   http://www.xilinx.com/support/documentation/boards_and_kits/zc702_zvik/2015_2/ug925-zynq-zc702-base-trd.pdf  
   http://www.wiki.xilinx.com/Zynq+Base+TRD+2015.2
   - :warning: For ZC702 Evaluation Board, not ZedBoard!
   - Remember to download the tutorial design files

## Known Issues (PetaLinux)

- Release Notes and Known Issues for PetaLinux 2013.04 and later (AR# 55776)
  - http://www.xilinx.com/support/answers/55776.html

- Ethernet
  - Insert the following into subsystems/linux/configs/device-tree/system-top.dts

  ```
  &gem0 {
  	phy-handle = <&phy0>;
  	ps7_ethernet_0_mdio: mdio {
  		#address-cells = <1>;
  		#size-cells = <0>;
  		phy0: phy@0 {
  			compatible = "marvell,88e1510";
  			device_type = "ethernet-phy";
  			reg = <0>;
  		} ;
  	} ;
  };
  ```

- UIO
  - Replace `compatible` for each GPIO device in subsystems/linux/configs/device-tree/pl.dtsi

  ```
  compatible = "generic-uio"
  ```

  - Replace `bootargs` in subsystems/linux/configs/device-tree/system-conf.dtsi
    - See https://github.com/Xilinx/linux-xlnx/commit/7ebd62dbc727ef343b07c01c852a15fc4d9cc9e5 for explanation.

  ```
  bootargs = "console=ttyPS0,115200 earlyprintk uio_pdrv_genirq.of_id=generic-uio";
  ```


## Miscellaneous (PetaLinux)

- Enable TCF agent
  - [rootfs] Filesystem Packages -> base -> tcf-agent

- Enable SSH server
  - [rootfs] Filesystem Packages -> console/network -> dropbear
  - [rootfs] Filesystem Packages -> console/network -> dropbear-openssh-sftp-server

- Mount via NFS
  - `mkdir /mnt/nfs`
  - `mount -o port=2049,nolock,proto=tcp -t nfs 192.168.1.1:<directory> /mnt/nfs`
  - can now execute "myapp" in build/linux/rootfs/apps/myapp/myapp


