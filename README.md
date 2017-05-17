# THIS BRANCH IS OBSOLETE, OUTDATED, NOT MAINTAINED, AND MOST IMPORTANTLY, NOT SUPPORTED ANYMORE!

# AOSP+OP-TEE manifest

This repository contains a local manifest that can be used to build an
AOSP build that includes OP-TEE for the hikey board.

## 1. References

* [AOSP Hikey build instructions][1]

## 2. Prerequisites

* Should already be able to build AOSP.  Distro should have necessary
  packages installed, and the repo tool should be installed.  Note
  that AOSP needs to be built with Java.  Also make sure that
  the `mtools` package is installed, which is needed to make the hikey
  boot image.

* In addition, you will need the pre-requisites necessary to build
  optee-os.

  After following the AOSP setup instructions, the following
  additional packages are needed.

```
sudo apt-get install android-tools-adb android-tools-fastboot autoconf \
	automake bc bison build-essential cscope curl device-tree-compiler flex \
	ftp-upload gdisk iasl libattr1-dev libc6:i386 libcap-dev libfdt-dev \
	libftdi-dev libglib2.0-dev libhidapi-dev libncurses5-dev \
	libpixman-1-dev libssl-dev libstdc++6:i386 libtool libz1:i386 make \
	mtools netcat python-crypto python-serial python-wand unzip uuid-dev \
	xdg-utils xterm xz-utils zlib1g-dev python-mako openjdk-8-jdk \
	ncurses-dev realpath android-tools-fsutils dosfstools libxml2-utils
```

## 3. Build steps

### 3.1. In an empty directory, clone the tree:

```
repo init -u https://android-git.linaro.org/git/platform/manifest.git -b android-8.1.0_r29 -g "default,-notdefault,-device,hikey"

# Please do NOT run below command! Internal reference only!
# repo init -u /home/ubuntu/aosp-mirror/platform/manifest.git -b android-8.1.0_r29 -g "default,-notdefault,-device,hikey" -p linux --depth=1
```

**!!! BIG WARNING !!!**: Do **NOT** to use `--depth=1` option, else patches in 3.4 will **FAIL** to apply!!! If you must use `--depth=1`, then you must find out which repos the patches apply to and `--unshallow` all of them! The number of repos are not few, and can increase over time.

## For relatively stable builds, use steps 3.2a-3.3a.

### 3.2a. Add the OP-TEE overlay:

```
cd .repo/manifests/
wget https://raw.githubusercontent.com/linaro-swg/optee_android_manifest/lcr-ref-hikey-o/pinned-manifest_YYYYMMDD.xml
cd ../../
```

**NOTE**: Replace `YYYYMMDD` with the date corresponding to the
`pinned-manifest_*.xml` files listed above. If there are build errors,
try an older manifest in the `archive/`, or use steps 3.2b-3.3b.

### 3.3a Sync

```
repo sync -m pinned-manifest_YYYYMMDD.xml
```

**WARNING**: Do **NOT** use -c option when sync-ing!

## For relatively recent builds, use steps 3.2b-3.3b.

### 3.2b. Add the OP-TEE overlay:

```
cd .repo
rm -f manifests/pinned-manifest*.xml
git clone https://android-git.linaro.org/git/platform/manifest.git -b linaro-oreo local_manifests
cd local_manifests
rm -f swg.xml
wget https://raw.githubusercontent.com/linaro-swg/optee_android_manifest/lcr-ref-hikey-o/swg.xml
cd ../../
```

### 3.3b. Sync

```
repo sync
repo manifest -r -o pinned-manifest-"$(date +%Y%m%d)".xml

# Please do NOT run below command! Internal reference only!
#repo manifest -r -o pinned-manifest-"$(date +%Y-%m-%d_%H:%M:%S)".xml
#./unshallow.sh
```

**WARNING**: Do **NOT** use -c option when sync-ing!

### 3.4. Apply the required patches (**please respect order!**)

**NOTE:** Apply the patches below **1 by 1** and make sure each patch is
applied successfully before applying the next one!

```
./android-patchsets/hikey-o-workarounds
./android-patchsets/O-RLCR-PATCHSET
./android-patchsets/hikey-optee-o
./android-patchsets/hikey-optee-4.9
./android-patchsets/OREO-BOOTTIME-OPTIMIZATIONS-HIKEY
./android-patchsets/optee-master-workarounds
./android-patchsets/swg-mods-o
```

**WARNING: If you run `repo sync` again at any time in the future to update
all the repos, by default all the patches above would be discarded, so you'll
have reapply them again before rebuilding!**

### 3.5. Download the build script

```
wget https://raw.githubusercontent.com/linaro-swg/optee_android_manifest/lcr-ref-hikey-o/build_aosp.sh
chmod +x build_aosp.sh
```

### 3.7. Build AOSP

For an 8GB board, use:
```
./build_aosp.sh
```

For a 4GB board, use:
```
./build_aosp.sh -4g
```

**WARNING**: Do **NOT** use `sudo`!

**WARNING: If you run `repo sync` again at any time in the future to update
all the repos, by default all the patches from 3.4 above would be discarded,
so you'll have reapply them again before rebuilding!**

**NOTE:** You can add the `-squashfs` option to make `system.img` size
smaller, but this will make `/system` read-only, so you won't be able to
push files to it.

## 4. Flashing the image

The instructions for flashing the image can be found in detail under
`device/linaro/hikey/install/README` in the tree.
1. Jumper links 1-2 and 3-4, leaving 5-6 open, and reset the board.
2. Invoke

```
cp -a out/target/product/hikey/*.img device/linaro/hikey/installer/hikey/
sudo ./device/linaro/hikey/installer/hikey/flash-all.sh /dev/ttyUSBn
```

where the ttyUSBn device is the one that appears after rebooting with
the 3-4 jumper installed.  Note that the device only remains in this
recovery mode for about 90 seconds.  If you take too long to run the
flash commands, it will need to be reset again.

## 5. Partial flashing

The last handful of lines in the `flash-all.sh` script flash various
images.  After modifying and rebuilding Android, it is only necessary
to flash *boot*, *system*, *cache*, and *userdata*.  If you aren't
modifying the kernel, *boot* is not necessary, either.

This directory contains a prebuilt trusted firmware image `fip.bin`.
If you wish to build the trusted os from source, follow the steps in the
**Build the bootloader firmware (fip.bin)** section above.

## 6. Running xtest

Please do NOT try to run `tee-supplicant` as it has already been started
automatically as a service! Once booted to the command prompt, `xtest`
can be run immediately.

## 7. Enable adb over usb

Boot the device. On serial console:

```
su setprop sys.usb.configfs 1
stop adbd
start adbd
```

[1]: https://source.android.com/source/devices.html
