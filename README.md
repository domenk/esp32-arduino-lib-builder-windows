# Compile custom arduino-esp32 libraries on Windows

If you are building ESP32 code using Arduino IDE, you can change some of the board settings using *Tools* menu, for example CPU frequency and flash frequency/mode/size. But if you want to change any other setting that `idf.py menuconfig` command allows you to (for example RTC source), you must build custom arduino-esp32 library and overwrite the one that is used by Arduino IDE.

We can do that with **Espressif's [ESP32 Arduino Lib Builder](https://github.com/espressif/esp32-arduino-lib-builder) using Ubuntu on VirtualBox**.

## Build arduino-esp32 on Windows using Ubuntu

This page will guide you through all steps needed to build custom arduino-esp32 on Windows from scratch.

Instructions last updated: **2021-12-23**, [arduino-esp32](https://github.com/espressif/arduino-esp32) version **2.0.2**.

### Add ESP32 board to Arduino IDE

* Install and open [Arduino IDE](https://www.arduino.cc/en/software).

* Go to *File* → *Preferences* and add the following URL to the *Additional Boards Manager URLs* field:

	`https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`

	(URL is taken from [Arduino ESP32 Installing](https://docs.espressif.com/projects/arduino-esp32/en/latest/installing.html).)

* Go to *Tools* → *Board* → *Boards Manager* and install *esp32* board, version *2.0.2*.

* Close Arduino IDE.

### Install VirtualBox and Ubuntu

* Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (*Windows hosts*).

* Download [Ubuntu Server](https://ubuntu.com/download/server) ISO image. Choose *Option 2 - Manual server installation* and download one of the releases. Instructions below are for version 21.10.

* In VirtualBox, go to *Machine* → *New*.
	* *Name and operating system*: Enter "Ubuntu" as *Name*. *Type* should be *Linux* and *Version* should be *Ubuntu (64-bit)*.
	* *Memory size*: Enter 2048 MB if you have enough resources.
	* *Hard disk*: Select *Create a virtual hard disk now*.
	* *Hard disk file type*: *VDI* is fine.
	* *Storage on physical hard disk*: Select *Fixed size* if you have enough resources.
	* *File location and size*: Set size to 20 GB or at least 15 GB.

* Start the machine you have just created.

* *Select start-up disk*: Select Ubuntu Server ISO image you have downloaded above.

* Ubuntu Server installer will boot up and guide you through installation.
	* Default settings are mostly ok.
	* Change keyboard layout if it does not match yours.
	* Choose full Ubuntu Server (not minimized).
	* When confirming data loss on disks selected for installation, you can safely continue, because in this case "disk" refers to the 15 GB virtual disk we have created above.
	* *Your server's name*: `ubuntu`
	* *Pick a username*: `ubuntu`
	* *SSH Setup*: Check option *Install OpenSSH server*.
	* After installation completes, select *Reboot Now*.
	* *Failed unmounting /cdroom* error may appear. Just press enter key to continue.

* When system boots, *ubuntu login* prompt will appear. Enter `ubuntu` (username we chose above).

* We will now configure Ubuntu and VirtualBox to allow SSH connection. SSH will make using command line easier (native support for copy and paste) and will later allow us to transfer arduino-esp32 files to Windows without installing VirtualBox Guest Additions. Enter the following commands:
	* `sudo apt-get update`
	* `sudo apt-get upgrade`
	* `sudo apt-get install openssh-server` (should already be installed, but just to make sure)
	* `sudo service ssh start`
	* `sudo systemctl status ssh` (to verify service is running – it should read *Active: active (running)*)
	* `sudo ufw allow ssh` (to add a firewall rule)
	* `sudo ufw enable` (to enable firewall)
	* `sudo ufw status` (check firewall status – it should read *22/tcp ALLOW*)
	* `ip r` (write down the IP address that is displayed after *src*, for example *10.0.2.15* – we will need it later)
	* `sudo shutdown 0`

* In VirtualBox, go to the machine and select *Settings*.
	* Go to *Network* → *Advanced* → *Port Forwarding*.
	* Add new rule: Protocol *TCP*, Host IP *127.0.0.1*, Host Post *2222*, Guest IP is the IP that `ip r` command displayed (for example *10.0.2.15*), Guest Port *22*.

* Start machine in headless mode: go to *Machine* → *Start* → *Headless Start*. Machine will run but no additional window will be opened. You can always go to *Machine* → *Show* to view machine's screen. (When closing this window, you should select *Continue running in the background*.)

* We should now be able to connect to the machine using SSH. We will use Windows command prompt (*cmd.exe*), but you can use Putty or any other SSH terminal.

* Run Windows command prompt (Win+R on keyboard, enter `cmd`). Then connect to the machine: `ssh -p 2222 ubuntu@127.0.0.1`

* You should now be connected to the machine.

### Install ESP32 Arduino Lib Builder

(Commands are taken from [ESP32 Arduino Lib Builder – Build on Ubuntu and Raspberry Pi](https://github.com/espressif/esp32-arduino-lib-builder#build-on-ubuntu-and-raspberry-pi) section.)

* `sudo apt-get install git wget curl libssl-dev libncurses-dev flex bison gperf python3 python3-pip python3-setuptools python3-serial python3-click python3-cryptography python3-future python3-pyparsing python3-pyelftools cmake ninja-build ccache`

	(Original instructions list Python packages with `python-` prefix instead of `python3-`.)

* `sudo pip install --upgrade pip`
* `cd`
* `git clone https://github.com/espressif/esp32-arduino-lib-builder`
* `cd esp32-arduino-lib-builder`
* `./tools/update-components.sh`
* `source ./tools/install-esp-idf.sh`
* `cd ~/esp32-arduino-lib-builder`

### Edit configuration and build the library

Now you can edit *sdkconfig* files using `idf.py menuconfig` command. Rename relevant *sdkconfig.esp32...* file to *sdkconfig*, run `idf.py menuconfig` and then rename it back. For example:
* `mv sdkconfig.esp32s2 sdkconfig`
* `idf.py menuconfig`
* `mv sdkconfig sdkconfig.esp32s2`

To build arduino-esp32 library, run:
* `./build.sh`

Build process will take some time and show various warnings. You can ignore them.

Miscellaneous:
* If `idf.py menuconfig` command results in *idf.py: command not found* error, use command with the full path: `~/esp32-arduino-lib-builder/esp-idf/tools/idf.py menuconfig`.
* Original arduino-esp32 *sdkconfig* files can be found in each subfolder of `%USERPROFILE%\AppData\Local\Arduino15\packages\esp32\hardware\esp32\2.0.2\tools\sdk\` on Windows.
* To speed up build process and build library only for some platforms, not for all three (ESP32, ESP32-S2 and ESP32-C3), run `nano build.sh` and edit line starting with `TARGETS=`.

### Transfer and install custom library

To transfer files from Ubuntu to Windows, download [pscp.exe](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html).

Navigate Windows command prompt to folder with *pscp.exe* and execute the following commands (first command ends with a dot):
* `pscp.exe -P 2222 -r ubuntu@127.0.0.1:/home/ubuntu/esp32-arduino-lib-builder/out .`
* `pscp.exe -P 2222 -r ubuntu@127.0.0.1:/home/ubuntu/esp32-arduino-lib-builder/components/arduino/cores out/cores`

Library files will be transfered to `out` folder.

(Before proceeding with the steps below, I suggest you make a backup of `%USERPROFILE%\AppData\Local\Arduino15\packages\esp32\hardware\esp32\2.0.2\` folder.)

Edit file `out\platform.txt`: replace line `tools.esptool_py.path` with the one from original file `%USERPROFILE%\AppData\Local\Arduino15\packages\esp32\hardware\esp32\2.0.2\platform.txt`.

Copy and overwrite all files from `out` folder to `%USERPROFILE%\AppData\Local\Arduino15\packages\esp32\hardware\esp32\2.0.2\`.

Run Arduino IDE and you should be able to build your project.
