# WARNING. Work in progress!

# Don't read or use this.

# Building your own Linux distribution for Raspberry Pi

In this series of exercises you will build a Linux distribution for Raspberry Pi. We will begin by setting up our build machine environment. After that we will build the reference implementation called Poky and run it with QEMU (an emulator). In the next step we will customize Poky by adding a layer containing a helloworld example recipe and image. In the final step we will obtain Raspberry Pi hardware support and build an image that you could write to an SD card and boot.

Please feel free to contact me if you find any mistakes. All feedback is welcome. 

##Prerequisites
### Advice
The Yocto Project documentation recommends that you have at least 50 gb of hard disk space available.

In the examples I will work from a folder called yocto in my home directory. Therefore if you see the following path

    /home/oscar/yocto

Please make sure to change it.

###A supported distribution:
In order to use Yocto you need to install a Linux distribution that is supported. You can find the full list here: 

http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#detailed-supported-distros

In the examples I will use Ubuntu 14.04 LTS.

###Install required packages
The following packages are required for Ubuntu.

    sudo apt-get install gawk wget git-core diffstat unzip texinfo gcc-multilib \
    build-essential chrpath socat libsdl1.2-dev xterm

If you are using any other supported distribution you’ll find the required packages here:

http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#required-packages-for-the-host-development-system

##Step 1: Building Poky and using QEMU
First we need to get the reference implementation Poky.

    git clone --branch jethro --depth 1 git://git.yoctoproject.org/poky
    cd poky

We then have to set up our build environment.

    source oe-init-build-env qemu-build

If you happen to close your terminal you will have to run the command above again. We are now ready to build an image.

    bitbake core-image-minimal

Note that it might take hours to build everything the first time. 

Now we are ready to run. 

    runqemu qemux86

__Note__ If you're running on a system without display you may have to use: __runqemu qemux86 nographic__

You __may__ have to provide your password when starting qemu. If that is the case you will see something like this:

    [sudo] password for oscar:

Wait for the system to boot and then login as __root__

    qemux86 login: root
    root@qemux86:~# uname -a
    Linux qemux86 4.1.17-yocto-standard #1 SMP PREEMPT Sun Apr 17 11:00:20 CEST 2016 i686 GNU/Linux

Congratulations! You have now built and executed Poky in QEMU.

##Step 2: Making our own layer and recipe
In the previous step we built Poky without any changes. In this step we're going to customize it by adding a layer that will contain an example recipe and our custom image.

Make sure that you're in the `poky` folder (for me that's `/home/oscar/yocto/poky`)

We will now generate our layer and example recipe. In the example I will call the layer `squeed` and the the recipe for `helloSqueed`. Please feel free to name yours whatever you like, but remeber to change the name in the examples that follow. 

    yocto-layer create squeed
    Please enter the layer priority you'd like to use for the layer: [default: 6] 10
    Would you like to have an example recipe created? (y/n) [default: n] y
    Please enter the name you'd like to use for your example recipe: [default: example] helloSqueed
    Would you like to have an example bbappend file created? (y/n) [default: n] n

    New layer created in meta-squeed.

    Don't forget to add it to your BBLAYERS (for details see meta-squeed\README).

You will notice that a folder `meta-squeed` has been created. The `meta-` is added as a naming prefix by convention. So the name of our layer is `meta-squeed`.

After adding the layer you should have folder with the following structure:

    tree meta-squeed/
    meta-squeed/
    ├── conf
    │   └── layer.conf
    ├── COPYING.MIT
    ├── README
    └── recipes-example
        └── example
            ├── helloSqueed-0.1
            │   ├── example.patch
            │   └── helloworld.c
            └── helloSqueed_0.1.bb

You will now have a generated recipe `helloSqueed_0.1.bb` with the source code: `helloworld.c` and `example.patch`

For fun lets edit `helloworld.c`

    nano meta-squeed/recipes-example/example/helloSqueed-0.1/helloworld.c

I change my example to print `Live long and prosper.\n` instead.

We also need an image that we will add our recipe to

    mkdir -p meta-squeed/recipes-core/images

Using nano (or your favorit editor) create the following image recipe     
     
    nano meta-squeed/recipes-core/images/qemu-squeed-image.bb

Enter the following information:

    require recipes-core/images/core-image-minimal.bb

    IMAGE_INSTALL += " helloSqueed"

__Note__ It's important to add a space in front of `helloSqueed`!

Go back to the `qemu-build` directory. For me that's `/home/oscar/yocto/poky/qemu-build`

    cd qemu-build
    
Now as a final step we need to add our new layer `meta-squeed` to our configuration file `conf/bblayers.conf`.
    
    bitbake-layers add-layer $HOME/yocto/poky/meta-squeed/

The last argument is the path to our created layer. __Make sure that it reflects the path that you have.__

If we take a look at `conf/bblayers.conf` you should see something like this:

You should see both the something like this 

    BBLAYERS ?= " \
      /home/oscar/yocto/poky/meta \
      /home/oscar/yocto/poky/meta-yocto \
      /home/oscar/yocto/poky/meta-yocto-bsp \
      /home/oscar/yocto/poky/meta-squeed \
      "

Alternatively we could also use `bitbake-layers` to inspect what layers we have added.

    layer                 path                                      priority
    ==========================================================================
    meta                  /home/oscar/yocto/poky/meta               5
    meta-yocto            /home/oscar/yocto/poky/meta-yocto         5
    meta-yocto-bsp        /home/oscar/yocto/poky/meta-yocto-bsp     5
    meta-squeed           /home/oscar/yocto/poky/meta-squeed        10


Now we can build our custom image by running:

    bitbake qemu-squeed-image

When the build has completed we can run it with QEMU as before.

    runqemu qemux86
    
When startup is completed you will see the following:

    Poky (Yocto Project Reference Distro) 2.0.1 qemux86 /dev/ttyS0

    qemux86 login: root
    root@qemux86:~# helloworld
    Hello IoT Tech Day!

##Step 3: Building our distribution for Raspberry Pi
In the final step we will now get our Linux distribution running on actual Raspberry Pi hardware. To do this we need a  __Board Support Package__ to provide hardware support. We can find this in the `meta-raspberryp` layer. We will also create another image to make our distribution run on hardware.

__Hint__ You can search for available layers [here](http://layers.openembedded.org/)

    cd .. 
    git clone -b jethro git://git.yoctoproject.org/meta-raspberrypi
    
If we check inside the meta-raspberrypi layer we will see that there are three available images.

    ls meta-raspberrypi/recipes-core/images/
    rpi-basic-image.bb  rpi-hwup-image.bb  rpi-test-image.bb

We will base our new image on the `rpi-basic-image`
    
    nano meta-iot-tech-day/recipes-core/images/rpi-iot-tech-image.bb
 
Make sure the file contains:        

    require /home/oscar/yocto/poky/meta-raspberrypi/recipes-core/images/rpi-basic-image.bb

    IMAGE_INSTALL += " helloIotTech"

__NOTE__ Make sure to replace the full path above!

Change directory to the `build` directory.

    cd build
    
We will then add the Raspberry Pi layer

    bitbake-layers add-layer $HOME/yocto/poky/meta-raspberrypi/
    
This means that your `conf/bblayers.conf` should now also have the `meta-raspberrypi` layer

    BBLAYERS ?= " \
      /home/oscar/yocto/poky/meta \
      /home/oscar/yocto/poky/meta-yocto \
      /home/oscar/yocto/poky/meta-yocto-bsp \
      /home/oscar/yocto/poky/meta-iot-tech-day \
      /home/oscar/yocto/poky/meta-raspberrypi \
      "

We are now ready to build our new image with hardware support. 

If you have a __Raspberry Pi 1__ use the following command

    MACHINE=raspberrypi bitbake rpi-iot-tech-image

Otherwise if you have Raspberry Pi 2:

    MACHINE=raspberrypi2 bitbake rpi-iot-tech-image

While running bitbake you should notice that the `Build Configuration` has changed. For example:

    Build Configuration:
    ...
    MACHINE           = "raspberrypi"
    ...
    meta              
    meta-yocto        
    meta-yocto-bsp    
    meta-iot-tech-day = "jethro:6dba9abd43f7584178de52b623c603a5d4fcec5c"
    meta-raspberrypi  = "jethro:f2cff839f52a6e6211337fc45c7c3eabf0fac113"

When the build is completed you'll find the bootable image here

    build/tmp/deploy/images/raspberrypi/rpi-iot-tech-image-raspberrypi.rpi-sdimg

Advice how to write this image to an SD Card can be found  [here](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md)

Once that is done you should be ready to go. If you want you can connect a monitor to it and after it's booted you should be able to ssh to it. In my case

    ssh root@192.168.0.17
    root@raspberrypi:~# helloworld 
    Hello IoT Tech Day!

It works and this completes our excercises. But we have only scratched the surface of what is possible. You could for example add more applications, write a kernel module, run Poky on some other hardware or investigate booting from TFTP. There are lots of more things to discover and I hope you'll have fun while doing so!

## More information
For more information I can warmly recommend reading the following resources.

[Yocto Project Quick Start](http://www.yoctoproject.org/docs/current/yocto-project-qs/yocto-project-qs.htlm)

[Search for layers and recipes](http://layers.openembedded.org/)

[About meta-raspberrypi layer](http://git.yoctoproject.org/cgit/cgit.cgi/meta-raspberrypi/about/)

[Yocto Project Documentation](https://www.yoctoproject.org/documentation) 

[Example recipe: mtr](http://cgit.openembedded.org/meta-openembedded/tree/meta-networking/recipes-support/mtr/mtr_0.86.bb)
