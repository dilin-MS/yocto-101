
# Build Yocto System for Raspberry Pi under Win10

## Purpose of this Document

Check the article [yocto-investigation](./yocto-investigation.md) to have a brief understanding of Yocto Project. This article is a tutorial to build your first yocto image under Win 10 and run the image on your Raspberry Pi3 hardware.

First we need to set up development environment for our host machine. We install docker since we are developing under Windows system. Then we install the yocto project SDK, modify some configurations and build our simple image. Finally we flash the image to sdcard so it can run on RPI.

### References

* [Yocto Project official site](https://www.yoctoproject.org)
* [Building Our First Poky Image for the Raspberry Pi](https://hub.packtpub.com/building-our-first-poky-image-raspberry-pi/)
* [Windows Instructions Docker Toolbox](https://github.com/crops/docker-win-mac-docs/wiki/Windows-Instructions-%28Docker-Toolbox%29)

### Platform

* Host machine: Win 10, x86-64 PC
* target machine: Raspberry Pi3 B+

## Set Up Development Environment for Host Machine

1. Download Docker for cross-compilation environment

    Concerning the host machine, naive Linux host is easier to start (see [Quick Start Guide for Systems Developers](https://www.yoctoproject.org/docs/2.6/brief-yoctoprojectqs/brief-yoctoprojectqs.html)). For Windows and Mac OS, we leverage docker to set up cross-compilation development environment (see [CROPS](https://www.yoctoproject.org/docs/2.6/dev-manual/dev-manual.html#setting-up-to-use-crops)). This exmaple demonstrates solution for windows10.

    Choose the right version of docker concerning your PC platform. We choose `Docker CE Stable` for Win 10. Download docker and install it.

2. Configurate docker-machine and create poky container

    2.1 Change default vm settings

    * Run powershell as administrator and run the following commands.

    * Remove the default docker-machine

        ```bash
        docker-machine rm default
        ```

    * Re-create the default docker-machine

        Win10 has inner `hyper-v` vitual machine driver, so we don't need to use the virtualbox in the reference. Enable hyper-v and prepare hyper-v following this [instructions](https://docs.docker.com/machine/drivers/hyper-v/).

        * Choose the number of cpus with `--hyperv-cpu-count`. For this example we'll use two.
        * Choose the amount of RAM: `--hyperv-memory`. This is also based on the host hardware. However, choose at least 4GB.
        * Choose the amount of disk space: `--hyperv-disk-size`. It is recommended that this be at least 50GB since building generates a lot of output. In this example we'll choose 50GB.
        * Create vm with new settings

            ```bash
            docker-machine create -d hyperv --hyperv-cpu-count=2 --hyperv-memory=4096 --hyperv-disk-size=50000 default
            ```
        * Restart docker

            Restart docker-machine to avoid some unexpected errors later on.
            ```bash
            docker-machine restart
            exit
            ```
            Use command `docker-machine ls` to list and check your settings.

    2.2 Create the samba container

    * Create a volume. A volume is used to persistent data on the host machine.

        ```bash
        docker volume create --name myvolume
        docker run -it --rm -v myvolume:/workdir busybox chown -R 1000:1000 /workdir
        ```
        The second command is executed to change the read-write permission of the `/workdir`.

    * create container samba to monitor poky container.

        ```bash
        docker create -t -p 445:446 --name samba -v myvolume:/workdir crops/samba
        docker start samba
        ```

    2.3 Create a Poky container

    Before using the poky container, make sure the samba container is running. Note that if you have started it in a previous terminal it will still be running. Run poky container as follows, note that we use the volume created above when specifying the workdir.

    ```bash
    docker run --rm -it -v myvolume:/workdir crops/poky --workdir=/workdir
    ```

    You will see a prompt that looks like

    ```bash
    pokyuser@892e5d2574d6:/workdir$
    ```

## Build Yocto for your raspberry pi

1. Go to [Raspberry Pi Official site](https://www.raspberrypi.org/documentation/hardware/raspberrypi/) to check the board model number.

    Choose the model number for your certain hardware. For Example, we have a Raspberry Pi 3B+ board. Thus the hardware in raspberry pi is `BCM2837B0`. We select `raspberrypi3` as the choice for `MACHINE`. See the conf file in `meta-raspberrypi/conf/machine` for details. Remenber this machine type, later you will need to fill it in the `local.conf` file. 

    Supported machine:

    * raspberrypi (BCM2835)
    * raspberrypi0 (BCM2835)
    * raspberrypi0-wifi (BCM2835)
    * raspberrypi2 (BCM2836 or BCM2837 v1.2+)
    * raspberrypi3 (BCM2837)
    * raspberrypi3-64 (64 bit kernel & userspace)
    * raspberrypi-cm (dummy alias for raspberrypi) (BCM2835)
    * raspberrypi-cm3 (dummy alias for raspberrypi2) (BCM2837)

    Note: The raspberrypi3 machines include support for Raspberry Pi 3B+.

    The example in this article use Yocto Project version 2.6 and poky version 2.6

2. Download the Poky metadata

    ```bash
    git clone git://git.yoctoproject.org/poky (branch master)
    ```

3. Download the Raspberry Pi BSP and dependencies metadata as layers

    ```bash
    cd poky
    git clone git://git.yoctoproject.org/meta-raspberrypi
    ```

    Poky is just a reference minimal system for us to start, we can add layers to customize our linux system specific to our hardware. Here we have a raspberry pi board. So we need the official BSP: `meta-raspberrypi` layer. What's more, according to [Yocto: meta-raspberrypi](https://git.yoctoproject.org/cgit/cgit.cgi/meta-raspberrypi/about/), there are other dependencies we need:

    * URI: git://git.openembedded.org/meta-openembedded
        * layers: meta-oe, meta-multimedia, meta-networking, meta-python
        * branch: master
        * revision: HEAD

    ```bash
    git clone git://git.openembedded.org/meta-openembedded
    ```

    Now we have `/meta-raspberrypi` and `/meta-openembedded` folders under `/poky` directory.

4. The oe-init-build-env script

    The Poky directory contains a script named `oe-init-build-env`. This is a script for the configuration/initialization of the build environment. It is not intended to be executed but must be *"sourced"*. Its work, among others, is to initialize a certain number of environment variables and place yourself in the build directory’s designated argument. The script must be run as shown here:

    ```bash
    source oe-init-build-env [build-directory]
    ```

    Here, *build-directory* is an optional parameter for the name of the directory where the environment is set (for example, we can use several build directories in a single Poky source tree); in case it is not given, it defaults to *build*. The *build-directory* folder is the place where we perform the builds. But, in order to standardize the steps, we will use the following command throughout to initialize our environment:

    ```bash
    $ source oe-init-build-env rpi-build

    ### Shell environment set up for builds. ###

    You can now run 'bitbake '
    Common targets are:
    core-image-minimal
    core-image-sato
    meta-toolchain
    adt-installer
    meta-ide-support
    You can also run generated qemu images with a command like 'runqemu qemux86'
    ```

    When we initialize a build environment, it creates a directory (the conf directory) inside rpi-build. This folder contains two important files:
    * *local.conf*: It contains parameters to configure BitBake behavior.
    * *bblayers.conf*: It lists the different layers that BitBake takes into account in its implementation. This list is assigned to the BBLAYERS variable.

5. Editing the local.conf file

    Search in the `/rpi-build/conf/local.conf` file. Find the `MACHINE ??=` line and change it to :

    ```bash
    MACHINE ??= "raspberrypi3"
    ```

    Here, the `"raspberrypi3"` is the machine type we found out before according to our board model number.

6. Editing the bblayers.conf file

    Add the `meta-raspberrypi` layer and `meta-oe`, `meta-multimedia`, `meta-networking`, `meta-python` dependences layers to the bblayers.conf file. The file should look like this:

    ```bash
    # POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
    # changes incompatibly
    POKY_BBLAYERS_CONF_VERSION = "2"
    BBPATH = "${TOPDIR}"

    BBFILES ?= ""

    BBLAYERS ?= " \
    /workdir/dilin/poky/meta \
    /workdir/dilin/poky/meta-poky \
    /workdir/dilin/poky/meta-yocto-bsp \
    /workdir/dilin/poky/meta-raspberrypi \
    /workdir/dilin/poky/meta-openembedded/meta-oe \
    /workdir/dilin/poky/meta-openembedded/meta-multimedia \
    /workdir/dilin/poky/meta-openembedded/meta-networking \
    /workdir/dilin/poky/meta-openembedded/meta-python \
    "
    ```

7. Building the Poky image

    * Choosing the image

        At this stage, we will have to look at the available images as to whether they are compatible with our platform (.bb files).
        Poky provides several predesigned image recipes that we can use to build our own binary image. We can check the list of available images by running the following command from the poky directory:

        ```bash
        ls meta*/recipes*/images/*.bb
        ```

        We want to build some raspberrypi-specific image. There are there of them: `rpi-test-image`, `rpi-hwup-image` and `rpi-basic-image`. According to [meta-raspberrypi docs](https://meta-raspberrypi.readthedocs.io/en/latest/layer-contents.html), the latter two images have been deprecated, so we build the `rpi-test-image` image. It is based on `core-image-base` which includes most of the packages in this layer and some media samples.

    * Running BitBake
    Under our build directory `rpi-build`, run the following command:

        ```bash
        pokyuser@49065a978b08:/workdir/poky/rpi-build$ bitbake rpi-test-image
        Loading cache: 100% |#########################################################################################################################################################################################################################################################################################| Time: 0:00:00
        Loaded 3164 entries from dependency cache.
        NOTE: Resolving any missing task queue dependencies

        Build Configuration:
        BB_VERSION           = "1.40.0"
        BUILD_SYS            = "x86_64-linux"
        NATIVELSBSTRING      = "universal"
        TARGET_SYS           = "arm-poky-linux-gnueabi"
        MACHINE              = "raspberrypi3"
        DISTRO               = "poky"
        DISTRO_VERSION       = "2.6+snapshot-20181203"
        TUNE_FEATURES        = "arm armv7ve vfp thumb neon vfpv4 callconvention-hard cortexa7"
        TARGET_FPU           = "hard"
        meta
        meta-poky
        meta-yocto-bsp       = "master:7faf6a00ba55db5cb5f2c21a72d09df8d784e9e8"
        meta-raspberrypi     = "master:c8a05f2fcf17ca2556f8a56086ea36da2e777d0f"
        meta-oe
        meta-multimedia
        meta-networking
        meta-python          = "master:ebc7b9e20ac22f6f2ad373621917f53e8a9af81c"

        Initialising tasks: 100% |####################################################################################################################################################################################################################################################################################| Time: 0:00:03
        Sstate summary: Wanted 187 Found 186 Missed 1 Current 1175 (99% match, 99% complete)
        NOTE: Executing SetScene Tasks
        NOTE: Executing RunQueue Tasks
        NOTE: Tasks Summary: Attempted 4006 tasks of which 4006 didn't need to be rerun and all succeeded.
        ```

    If encountered with the below error message, add a line `LICENSE_FLAGS_WHITELIST = "commercial"` in *rpi-build/conf/local.conf* file.

    > ERROR: Nothing PROVIDES 'libav' (but /workdir/poky/meta-raspberrypi/recipes-multimedia/omxplayer/omxplayer_git.bb DEPENDS on or otherwise requires it)ffmpeg PROVIDES libav but was skipped: because it has a restricted license 'commercial'. Which is not whitelisted in LICENSE_FLAGS_WHITELIST

    The first bitbake can take quite a long time. Elapsed time varies concerning the layers you add. For reference, it takes 4h for the first bitbake of `core-image-minimal` image. And over 8h to bitbake `rpi-test-image` image.

## Flash Image to SDcard for Raspberry Pi

1. Copy image from docker to your PC

    After hours of baking, we can rejoice with the result and the creation of the system image for our target:

    ```bash
    $ ls rpi-build/tmp/deploy/images/raspberrypi3/*sdimg
    rpi-test-image-raspberrypi3-20181203014212.rootfs.rpi-sdimg  rpi-test-image-raspberrypi3.rpi-sdimg
    ```

    `rpi-test-image-raspberrypi3-20181203014212.rootfs.rpi-sdimg` can be used with Win32DiskImager in Windows, while in Linux you can use the `dd` command with the `rpi-test-image-raspberrypi3.rpi-sdimg` directly to flash image to SD card.

    Open another powershell terminal and run:

    ```bash
    docker cp 6446fd34ba5e:/workdir/poky/rpi-build/tmp/deploy/images/raspberrypi3/rpi-test-image-raspberrypi3-20181203014212.rootfs.rpi-sdimg [directory in your windows] // Replace 6446fd34ba5e with your samba container ID
    ```

2. Flash image to SD card

    See [Raspberry pi doc](https://www.raspberrypi.org/documentation/installation/sdxc_formatting.md).

    * Format SD card first

        Plug in your SD card and you can see two disk device concerning your SD card. One is named `boot`, another is USB drive. Download SD Format and format the `boot` device.

        Format SD-card before flashing. Or you propabably will get a `"Error 5 access denied error"` when trying to write an image to your SD-card.

    * Flash image to SD card

        Download Win32DiskImager(See [raspberry pi instructions](https://www.raspberrypi.org/documentation/installation/installing-images/windows.md)). From Win32DiskImager choose the `rpi-test-image-raspberrypi3-20181203014212.rootfs.rpi-sdimg` from the directory you have just saved it, and choose the `boot` disk device.
        Carefully check you choose the right device and click write.

3. After you successfully start the system on raspberry pi, you see:

    ```bash
    raspberrypi3 login:
    ```

    Input `root` and you can use the system now.

    HURRAY!!!

## Troubleshooting

* If you cannot bind local 445 port to samba's 445 port, you can use the below command to validate everything goes well in pocky. Open another powershell terminal as administrator and run the below docker command to help debug.

    ```bash
    PS C:\windows\system32> docker exect -it [your samba container ID] bash // go into samba container
    root@a6fd96ce2b55:# cd workdir/
    ```

* When BitBake fails for some package uninstalled, open another powershell terminal as administrator and run the below docker command to help debug.

    ```bash
    PS C:\windows\system32> docker exec -it -u 0 [your poky container ID] bash
    root@a6fd96ce2b55:# sudo apt-get install [your needed package name]
    ```

* Open another powershell terminal as administrator and run the below docker command to check image's history actions.

    ```bash
    docker history crops/poky
    ```

* If you encounter BitBake error with messages showed below, delete the `/rpi-build/tmp` folder to free some space. Run command `du -sh ./tmp` under `/rpi-build` directory to see the size of tmp folder. Or you can open another powershell terminal as administrator and run the following command to check the size of the build directory.

    ```bash
    docker exec -it samba bash
    cd workdir/poky/rpi-build
    du-h --max-depth=1
    ```

* If you plug in your sd-card which has been succefully wrote the rpi-test-image, and the system boots continuously prints out:

    ```bash
    init: id "s0" respawning too fast: disabled for 5 minutes
    ```

    Go back and add `ENABLE_UART = "1"` in local.conf. Re-bake your image and the problem can be solved.

* Useful docker command:

    ```bash
    docker image ls // list all images
    docker container ls // list all running containers
    docker container ls --all // list all containers
    docker container stop [NAME or ID] // stop certain container
    docker container rm [NAME or ID] // remove certain container(container should be stopped first before removed)
    ```
    See more in [Docker tutorial](https://docs.docker.com/get-started/).