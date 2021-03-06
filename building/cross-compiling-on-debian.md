# Setting up a Debian system to build a GHC cross-compiler



This is a (possibly incomplete) guide to setting up a Debian system to allow building a cross compiler. This is known to work for building x86\_64 to armhf and x86\_64 to arm64 cross compilers. Others architectures may also work. For the purposes of this documentation we will assume that we want to cross-compiler for both 32 bit armhf and 64 bit arm63/aarch64.



A good first step is to make sure that your machine is set up the build a native GHC. The easiest way to do this is to run:


```wiki
sudo apt-get build-dep ghc
```


This installs all the packages need to build a the native GHC Debian package.



Now its time to set up Debain multi-arch which allows you to install Debian packages for more than one architecture on a single machine. The Debian documention for this is at [
https://wiki.debian.org/Multiarch/HOWTO](https://wiki.debian.org/Multiarch/HOWTO) .



Once that is set up you should be able to query the available architectures using:


```wiki
dpkg --print-foreign-architectures
```


That command should list the new foreign architectures.



Then make sure that the machine's local apt database gets updated using:


```wiki
sudo apt-get update
```


The next step is to install the required C compilers, binutils and libraries required to build the GHC cross compilers. First for arm64/aarch64:


```wiki
sudo apt-get install binutils-aarch64-linux-gnu cpp-5-aarch64-linux-gnu gcc-5-aarch64-linux-gnu
sudo apt-get install linux-libc-dev-arm64-cross libc6-arm64-cross libc6-dev:arm64
sudo apt-get install libtinfo-dev:arm64 libncurses5:arm64 libncurses5-dev:arm64 
```


and then for armhf:


```wiki
sudo apt-get install binutils-arm-linux-gnueabihf cpp-5-arm-linux-gnueabihf gcc-5-arm-linux-gnueabihf
sudo apt-get install linux-libc-dev-armhf-cross libc6-armhf-cross libc6-dev:arm64
sudo apt-get install libtinfo-dev:armhf libncurses5:armhf libncurses5-dev:armhf
```


Finally, to build for both arm64 and armhf, you need to install the LLVM tools. To build GHC 7.10 you will need llvm-3.5 while to build GHC 8.0 you will need llvm-3.7. Its ok to install both using:


```wiki
sudo apt-get install llvm-3.5 llvm-3.7
```


And thats it! Assuming you already have the GHC source distribution of Git checkout, build a cross compiler is as simple as changing into the top level directory of the sources and doing (`arm-linux-gnueabihf` can be replaced with `aarch64-linux-gnu`) :


```wiki
sed 's/#.* = quick-cross/BuildFlavour = quick-cross/' mk/build.mk.sample > mk/build.mk
perl boot
./configure --target=arm-linux-gnueabihf --prefix=$HOME/GHC/Cross/
make -j4
make install
```


Assuming everything succeeded, you will now have your cross compiler in `$HOME/GHC/Cross/`.


