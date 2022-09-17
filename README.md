# Compile custom arduino-esp32 libraries on Windows

If you are building ESP32 code using Arduino IDE, you can change some of the board settings using *Tools* menu, for example CPU frequency and flash frequency/mode/size. But if you want to change any other setting that `idf.py menuconfig` command allows you to (for example RTC source), you must build custom arduino-esp32 library and overwrite the one that is used by Arduino IDE.

We can do that with **Espressif's [ESP32 Arduino Lib Builder](https://github.com/espressif/esp32-arduino-lib-builder) using Ubuntu on VirtualBox**.

## Build arduino-esp32 on Windows using Ubuntu

This page will guide you through all steps needed to build custom arduino-esp32 on Windows from scratch.

Instructions last updated: **2022-09-17**, [arduino-esp32](https://github.com/espressif/arduino-esp32) version **2.0.5**.

### Add ESP32 board to Arduino IDE

* Install and open [Arduino IDE](https://www.arduino.cc/en/software).

* Go to *File* → *Preferences* and add the following URLs to the *Additional Boards Manager URLs* field:

	`https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
	`https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_dev_index.json`

	(URLs are taken from [Arduino-ESP32 Installing](https://docs.espressif.com/projects/arduino-esp32/en/latest/installing.html).)

* Go to *Tools* → *Board* → *Boards Manager* and install *esp32* board, version *2.0.5*.

* Close Arduino IDE.

### Install VirtualBox and Ubuntu

* Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (*Windows hosts*).

* Download [Ubuntu Server](https://ubuntu.com/download/server) ISO image. Instructions below are for version 22.04.1.

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
	* On *Storage configuration* step installer might not allocate all available disk space for mount point `/` (for example, you created disk with 20 GB, but installer allocated only 10 GB). Change the size of device mounted to `/` (*ubuntu-lv*) to fix this.
	* When confirming data loss on disks selected for installation, you can safely continue, because in this case "disk" refers to the 15 GB virtual disk we have created above.
	* *Your server's name*: `ubuntu`
	* *Pick a username*: `ubuntu`
	* *SSH Setup*: Check option *Install OpenSSH server*.
	* After installation completes, select *Reboot Now*.
	* Error *Failed unmounting /cdroom* may appear. Just press enter key to continue.

* When system boots, *ubuntu login* prompt will appear (if it does not appear but system seems to finish loading, press enter key and prompt should appear). Enter `ubuntu` (username we chose above).

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

(Commands are taken from [Arduino-ESP32 documentation – Library Builder](https://docs.espressif.com/projects/arduino-esp32/en/latest/lib_builder.html). Documentation on [ESP32 Arduino Lib Builder](https://github.com/espressif/esp32-arduino-lib-builder/blob/master/README.md) is obsolete.)

* `sudo apt-get install git wget curl libssl-dev libncurses-dev flex bison gperf cmake ninja-build ccache jq python3 python3-pip python-is-python3`
* `sudo pip install --upgrade pip`
* `sudo pip install --upgrade setuptools pyserial click cryptography future pyparsing pyelftools`
* `cd`
* `git clone https://github.com/espressif/esp32-arduino-lib-builder`
* `cd esp32-arduino-lib-builder`
* `./build.sh -t none`

(Last command takes a while to execute on the first run.)

If the last command prints *The following Python requirements are not satisfied* somewhere in the ouput, it failed. Run command `sudo pip install`, followed by packages listed in the output. Enclose package names in quotation marks and separate them with a space. The following command should probably do the trick:

`sudo pip install "pyparsing>=2.0.3,<2.4.0" "idf-component-manager~=1.0" "gdbgui==0.13.2.0" "pygdbmi<=0.9.0.2" "python-socketio<5" "itsdangerous<2.1" "kconfiglib==13.7.1" "reedsolo>=1.5.3,<=1.5.4" "bitstring>=3.1.6" "ecdsa>=0.16.0" "construct==2.10.54"`

If previous `pip install` command fails with error regarding *cffi* package version mismatch, run `sudo apt-get install python3-cffi` and then execute above `pip install` command again.

### Edit configuration

Changing build configuration (`sdkconfig` file) is a bit challenging. Because resulting `sdkconfig` file is concatenated from various files in `configs/` directory, we first need to generate "original" sdkconfig, then sdkconfig with wanted configuration changes, and at last incorporate these changes into files in `configs/` directory.

In the following commands we will use `esp32s2` as a target. Supported targets are: `esp32`, `esp32s2`, `esp32s3`, `esp32c3`.

* `./build.sh -b menuconfig -t esp32s2`
* Do not make any changes yet. Just quit.
* `cp sdkconfig sdkconfig.original`
* `./build.sh -b menuconfig -t esp32s2`
* Make configuration changes you want. Save and quit.
* `diff --color sdkconfig.original sdkconfig`

Resulting changes need to be incorporated into file `configs/defconfig.esp32s2`. Various tips:
* Some of the removed configuration options may not exist in `defconfig.esp32s2`. No need to do anything.
* You can skip added/removed lines that are commented (they start with "#").
* You can skip added deprecated options (from line "# Deprecated options for backward compatibility" to "# End of deprecated options").

After you've incorporated all changes, run `./build.sh -b menuconfig -t esp32s2` again and check if changes are applied.

### Build the library

Build for all targets:
* `./build.sh`

Build for a specific target only:
* `./build.sh -t esp32s2`

Note: script deletes old build files on each run.

### Transfer and install custom library

To transfer files from Ubuntu to Windows, download [pscp.exe](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html).

Navigate Windows command prompt to folder with *pscp.exe* and execute the following commands (first command ends with a dot):
* `pscp.exe -P 2222 -r ubuntu@127.0.0.1:/home/ubuntu/esp32-arduino-lib-builder/out .`
* `pscp.exe -P 2222 -r ubuntu@127.0.0.1:/home/ubuntu/esp32-arduino-lib-builder/components/arduino/cores out/cores`

Library files will be transfered to `out` folder.

(Before proceeding with the steps below, I suggest you make a backup of `%USERPROFILE%\AppData\Local\Arduino15\packages\esp32\hardware\esp32\2.0.5\` folder.)

Edit file `out\platform.txt`: replace line `tools.esptool_py.path` with the one from original file `%USERPROFILE%\AppData\Local\Arduino15\packages\esp32\hardware\esp32\2.0.5\platform.txt`.

Copy and overwrite all files from `out` folder to `%USERPROFILE%\AppData\Local\Arduino15\packages\esp32\hardware\esp32\2.0.5\`.

Run Arduino IDE and you should be able to build your project.
