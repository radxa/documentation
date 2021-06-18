<!---
---
title: Build AOSP Source for Radxa-zero
permalink: https://github.com/radxa/documentation/tree/master/rs102/build/aosp.md
---
-->

# Build AOSP Source for Radxa-zero



## How to get code
* Install git
```bash
sudo apt-get install git
```
* Config git
```bash
git config --global user.name "your user name"
git config --global user.email "your email"
```
* Get repo
```bash
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
echo "export PATH=~/bin:\$PATH" >> ~/.bashrc
```
* Repo init & repo sync
```bash
mkdir ~/W2 && cd ~/W2 # make a directory (etc. ~/w2) to save code
repo init -u ssh://git@gitlab.com/amlogic-android/manifests.git -b s905x3-android-p -m p-amlogic-openlinux-20191101-ott-aosp.xml
repo sync
```

### ENV

```bash
FROM ubuntu:xenial

RUN rm /etc/apt/sources.list
RUN echo "deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse" | tee /etc/apt/sources.list
RUN echo "deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse" >> /etc/apt/sources.list

RUN apt-get update -y && apt-get install -y openjdk-8-jdk python git-core gnupg flex bison gperf build-essential \
    zip curl liblz4-tool zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386\
    lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev ccache \
    libgl1-mesa-dev libxml2-utils xsltproc unzip mtools u-boot-tools \
    htop iotop sysstat iftop pigz bc device-tree-compiler lunzip \
    dosfstools vim-common parted udev

RUN curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo > /usr/local/bin/repo && \
    chmod +x /usr/local/bin/repo && \
    which repo

ENV REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/'
ENV PS1="\[\033[01;37m\]\u@build\[\033[00m\]:\[\033[01;32m\]\w\[\033[00m\]:$ "

RUN apt-get install -y python-pip && pip install pycrypto
RUN apt-get install -y lzop swig
RUN git config --global user.email docker9@vamrs.com && git config --global user.name docker9
COPY ./gcc-linaro-6.3.1-2017.02-x86_64_arm-linux-gnueabihf /opt/gcc-linaro-6.3.1-2017.02-x86_64_arm-linux-gnueabihf
COPY ./gcc-linaro-aarch64-none-elf-4.8-2013.11_linux /opt/gcc-linaro-aarch64-none-elf-4.8-2013.11_linux
RUN apt-get install -y net-tools gcc-arm-linux-gnueabihf gcc-arm-none-eabi
```



## How to build image
* use quick_build:

```bash
$ source ./build/envsetup.sh
$ lunch faraday-userdebug
$ ./device/amlogic/common/quick_compile.sh
[NUM]   [        PROJECT]   [       SOC TYPE]  [  HARDWARE TYPE]
---------------------------------------------
....
[  5]   [        Faraday]  [         S905Y2]  [           U223]
....
---------------------------------------------
# select 5 to choose S905Y2
please select platform type (default 1(Ampere)):5
Select compile Android verion type lists:
 [NUM]   [Android Version]
 [  1]   [AOSP]
 [  2]   [ DRM]
 [  3]   [GTVS](need google gms zip)
 --------------------------------------------
# select 1
Please select Android Version (default 1 (AOSP)):1
# output patch: out/target/product/faraday/aml_upgrade_package.img
```

* build manually:

```bash
# build bootloader
cd bootloader/uboot-repo
./mk g12a_u223_v1 --systemroot
cp build/u-boot.bin ../../device/amlogic/faraday/bootloader.img
cp build/u-boot.bin.usb.bl2 ../../device/amlogic/faraday/upgrade/u-boot.bin.usb.bl2
cp build/u-boot.bin.usb.tpl ../../device/amlogic/faraday/upgrade/u-boot.bin.usb.tpl
cp build/u-boot.bin.sd.bin ../../device/amlogic/faraday/upgrade/u-boot.bin.sd.bin
cd -

# build update image
source ./build/envsetup.sh
lunch faraday-userdebug
export PATH=/opt/gcc-linaro-6.3.1-2017.02-x86_64_arm-linux-gnueabihf/bin:$PATH
export PATH=${ANDROID_BUILD_TOP}/bootloader/uboot-repo/bl33/build/tools:$PATH
make otapackage
# output patch: out/target/product/faraday/aml_upgrade_package.img
```
* Partition build

After built whole image, if you want to rebuild some of partitions, there are commands:

boot.img
```bash
make bootimage
# to get boot.img at out/target/product/faraday/boot.img
```
logo.img
```bash
make logimg
```
recovery.img
```bash
make recoveryimage
```
system.img
```bash
make systemimage
```
dt.img
```bash
make dtbimage
```
u-boot.bin
```bash
cd bootloader/uboot-repo
./mk g12a_u223_v1 --systemroot
```
vendor.img
```bash
make vendorimage
```
odm.img
```bash
make odm_image
```

## How to burn image

### For Linux:
#### Prepare work

```bash
#git clone flash tool
git clone https://github.com/radxa/aml-flash-tool.git
#install dependency
cd aml-flash-tool
./INSTALL
```

#### Run Radxa-Zero into maskrom mode

Press and hold the button on the back side of Radxa-Zero before you plug in usb cable in the USB-PWR port.

If Radxa-Zero running in maskrom mode correctly, you will see following info when you execute lsusb command.

```log
$ lsusb
Bus 001 Device 052: ID 1b8e:c003 Amlogic, Inc. USB DEVICE
```

#### Burning the whole image

```bash
aml-flash-tool.sh aml_upgrade_package.img
```


