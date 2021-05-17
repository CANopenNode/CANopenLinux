CANopenLinux                                               {#readmeCANopenLinux}
============

CANopenLinux is a CANopen stack running on Linux devices.

It is based on [CANopenNode](https://github.com/CANopenNode/CANopenNode), which is free and open source CANopen Stack and is included as a git submodule.

CANopen is the internationally standardized (EN 50325-4) ([CiA301](http://can-cia.org/standardization/technical-documents)) CAN-based higher-layer protocol for embedded control system. For more information on CANopen see http://www.can-cia.org/.

CANopenLinux homepage is https://github.com/CANopenNode/CANopenLinux


Getting or updating the project
-------------------------------
Clone the project from git repository and get submodules:

    git clone https://github.com/CANopenNode/CANopenLinux.git
    cd CANopenLinux
    git submodule init
    git submodule update

Update the project:

    cd CANopenLinux
    git pull # or: git fetch; inspect the changes (gitk); git merge
    git submodule update


Usage
-----
Support for CAN interface is part of the Linux kernel, so called [SocketCAN](https://en.wikipedia.org/wiki/SocketCAN). CANopenNode runs on top of SocketCAN, so it should be able to run on any Linux machine, it depends on configuration of the kernel. Examples below was tested on Debian based machines, including Ubuntu and Raspberry PI. It is possible to run tests described below without real CAN interface, because Linux kernel already contains virtual CAN interface.

Windows or Mac users, who don't have Linux installed, can use [VirtualBox](https://www.virtualbox.org/) and install [Ubuntu](https://ubuntu.com/download/desktop) or similar. To get confortable you may like to enroll [Introduction to Linux](https://training.linuxfoundation.org/training/introduction-to-linux/).


### CAN interfaces
#### Virtual CAN interface
It can connect multiple programs inside Linux machine. It can be activated by the following commands:

    sudo modprobe vcan
    sudo ip link add dev can0 type vcan
    sudo ip link set up can0

#### USB, PCI or similar CAN interface
There are several CAN interfaces on the market which works with Linux SocketCAN. See [Linux kernel source](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/net/can), Kconfig files, for supported interfaces by the Linux kernel. For example [EMS CPC-USB](https://www.ems-wuensche.com/?post_type=product&p=746) or [PCAN-USB FD](http://www.peak-system.com/PCAN-USB-FD.365.0.html?&L=1). Usually such interface is started with:

    sudo ip link set up can0 type can bitrate 250000

#### Serial slcan interface
Most cheap CAN interface, for example [USBtin](http://www.fischl.de/usbtin/). It may not be fast enough and may lose messages. Usually it is started with (-s5=250kbps):

    sudo slcand -f -o -c -s5 /dev/ttyACM0 can0
    sudo ip link set up can0

#### CAN capes for Raspberry PI or similar
RPI may work wit some of the USB CAN interfaces or with CAN shield like [this](https://www.sg-electronic-systems.com/ecommerce/12-can-bus-shield). CAN shield may not be fast enough and may lose messages.


### CAN utilities
[SocketCAN userspace utilities and tools](https://github.com/linux-can/can-utils) contains several useful command line tools for CAN in Linux. Install with:

    sudo apt-get install can-utils

In own terminal run candump to display all CAN messages with timestamp:

    candump -td -a can0


### Running CANopenLinux device
Compile:

    cd CANopenLinux
    make

Install (copy `canopend` application to the `/usr/bin/` directory):

    sudo make install

Display options:

    canopend --help

Run on can0 device with CANopen NodeID = 4:

    canopend can0 -i 4

If NodeID is not specified, then CANopen LSS protocol may be used. Program can be finished by pressing Ctrl+c or with CANopen reset node command.

After connecting the CANopen Linux device into the CAN(open) network, bootup message is visible. By default device uses Object Dictionary from `CANopenNode/example`, which contains only communication parameters. With the external CANopen tool all parameters can be accessed and CANopen Linux device can be configured (For example write heartbeat producer time in object 0x1017,0).

When CANopen Linux device is first connected to the CANopen network it shows bootup message and emergency message, because of missing storage files. To avoid emergency message it is necessary to trigger saveAll command (write correct code into parameter 0x1010,1 with SDO command) and restart the program.

Note also, if there are multiple instances of canopend running from the same directory, storage path should be specified for each.


### CANopen ASCII command interface
CANopenNode includes CANopen ASCII command interface (gateway) specified by standard CiA309-3. It can be used as a commander for other CANopen devices: NMT master, LSS master, SDO client, etc. In CANopen Linux device command interface is available by default.

To use ASCII command interface on canopend directly just run it with `-c "stdio"` and type the commands followed by enter in it.

    canopend can0 -i 1 -c "stdio"
    help
    1 write 0x1010 1 vs save
    1 reset node

To create CANopen Linux commander device on local socket run:

    canopend can0 -i 1 -c "local-/tmp/CO_command_socket"

#### cocomm
CANopenLinux/cocomm directory contains a small command line program, which establishes socket connection with `canopend` (CANopen Linux commander device). It sends standardized CANopen commands (CiA309-3) to gateway and prints the responses to stdout and stderr. See [cocomm/README.md](cocomm/README.md) for usage.


Creating new project
--------------------
`canopend` is a basic CANopen Linux device with optional commander functionalities. However, CANopen device is able to be much more, like (simple) input/output (digital or analog) device according to standard CiA401 or anything else as specified by other CANopen device profiles or own idea.

New project can be started in new directory simply by adding customized makefile, custom Object Dictionary OD.h/c files and custom application source files in Arduino style, which are called from CO_main_basic.c file.

### Single or multi threaded application
By default canopend runs in single thread (CO_SINGLE_THREAD option in Makefile). Different events, such as can reception or timer expiration trigger looping through the stack (all code is non-blocking). It requires less system resources.

In multi threaded operation a real-time thread is established besides mainline thread. RT thread runs each millisecond and processes PDOs and optional application code with peripheral read/write, control program or similar. With this configuration race conditions must be taken into account, for example application code running from mainline thread must use CO_(UN)LOCK_OD macros when accessing OD variables.

See also [CANopenDemo](https://github.com/CANopenNode/CANopenDemo) for examples.


### Create new project with KDevelop
- https://www.kdevelop.org/
- `sudo apt install kdevelop breeze`
- Run KDevelop, select: Project -> open project
- Navigate to project directory and click open.
- KDevelop will recognize `Makefile` and will just use it. Click Finish.
- Open project settings (right click on project on left panel)
  - Make: set to 1 thread operation.
  - Language support, Includes, add paths to directories:
    - `<path_to_CANopenLinux_driver_files>`
    - `<path_to_CANopenNode>`
    - `<path_to_project_files>`
  - Language support, Defines, add:
    - `CO_DRIVER_CUSTOM`
- Run -> Setup launches -> `our_program`:
  - Add Executable file and name it.
  - Executable file: `<select_executable>`
  - Arguments: `can0 -i 4`
- Build, then Execute or Debug


Change Log
----------
- **[v4.0](https://github.com/CANopenNode/CANopenLinux/tree/HEAD) - current**: Git repository started on GitHub.
  - Linux driver files copied from https://github.com/CANopenNode/CANopenNode/tree/76b43c88ef6d5490cb2f1518e10646e8dcb45c76/socketCAN
  - cocomm copied from https://github.com/CANopenNode/CANopenSocket/tree/71f21e41fd4527718d8b6161938b479386d2d03b/cocomm
  - Submodule CANopenNode added.
  - Few adjustments and documentation written.


License
-------
This file is part of CANopenNode, an opensource CANopen Stack.
Project home page is <https://github.com/CANopenNode/CANopenNode>.
For more information on CANopen see <http://www.can-cia.org/>.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
