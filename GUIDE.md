# Building TensorFlow for Orange Pi Zero from source 

## Index 
1. [What you need](#what-you-need)
2. [Installing basic dependencies](#installing-basic-dependencies)
3. [Installing USB Memory as Swap](#installing-usb-memory-as-swap)
4. [Compiling Bazel](#compiling-bazel)
5. [Compiling TensorFlow](#compiling-tensorflow)
6. [Cleaning Up](#cleaning-up)
7. [References](#references)
8. [Some tips](#some-tips)

## What you need

* Orange Pi Zero (512MB version)
* A SD card (minimum of 16GB is recomended) with Armbian image
* Internet connection to Orange Pi Zero
* A USB memory drive (If you are going to use a flash drive, please read [tips](#some-tips) section to see what type of flash drive is better).
* A fair amount of time 

## Installing basic dependencies

First
```
sudo add-apt-repository ppa:webupd8team/java
sudo apt update
```

Then install Bazel dependencies 
```
sudo apt install pkg-config zip g++ zlib1g-dev unzip gcc-4.8 g++-4.8 build-essential oracle-java8-installer oracle-java8-set-default
```

And then install TensorFlow dependencies depending on which version of python you want to use :

```
# For Python 2.x
sudo apt-get install python-pip python-numpy swig python-dev
sudo pip install wheel

# For Python 3.x
sudo apt-get install python3-pip python3-numpy swig python3-dev python3-wheel
sudo pip3 install wheel
```
To be able to take advantage of certain optimization flags:
```
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 100
```
Finally, for cleanliness, make a directory that will hold the Protobuf, Bazel, and TensorFlow repositories.
```
mkdir temp
cd temp
```
## Installing USB Memory as Swap

The compiling procedure of TensorFlow on Orange Pi Zero is quite similar to that on a Raspberry Pi board. The only difference is in the amount of RAM these boards have, as Orange Pi Zero has only 512MB memory, the effect of swap space is a lot more significant during the compiling procedure and this can cause many troubles during compile of both Bazel and TensorFlow.

For swap space you need a USB drive with at least 1GB of memory for some tips on choosing this drive you can go to [tips](#some-tips) section. After choosing your driver, insert your USB drive, and find the `/dev/XXX` path for the device.

```shell
sudo blkid
```

It could be something like `/dev/sda1`

Then format your device to be swap:

```shell
sudo mkswap /dev/XXX
```

If the previous command outputted an alphanumeric UUID, copy that now. Otherwise, find the UUID by running `blkid` again. Copy the UUID associated with `/dev/XXX`

```shell
sudo blkid
```

Now edit your `/etc/fstab` file to register your swap file and increase the size of your tmpfs

```shell
sudo nano /etc/fstab
```

On a separate line, enter the following information. Replace the X's with the UUID (without quotes)

```bash
UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX none swap sw,pri=5 0 0
```

Add the following to the end of the tmpfs line
```bash
defaults,nosuid,size=4G 0 0
```

Save `/etc/fstab`, exit your text editor, and run the following command:

```shell
sudo swapon -a
```
## Compiling Bazel

To build [Bazel](https://github.com/bazelbuild/bazel), we're going to need to download a zip file containing a distribution archive. Let's do that now and extract it into a new directory called `bazel`:

```shell
wget https://github.com/bazelbuild/bazel/releases/download/0.11.1/bazel-0.11.1-dist.zip
unzip -d bazel bazel-0.11.1-dist.zip
sudo chmod -R 777 bazel/
```

Once it's done downloading and extracting, we can move into the directory to make a few changes:

```shell
cd bazel
```

Now for preventing from any out of memory error :

```shell
sed -i -e 's/-encoding UTF-8 "@${paramfile}"/-encoding UTF-8 "@${paramfile}" -J-Xmx500M/g' scripts/bootstrap/compile.sh
```

Let's say to Bazel that our CPU is ARM by adding return "arm" in get_cpu_value function before trying to find it from OS name :
```shell
sed -i '95i\  return "arm"' tools/cpp/lib_cc_configure.bzl
```

Now we can build Bazel! _Note: this also takes some time._

```shell
sudo ./compile.sh
```

When the build finishes, you end up with a new binary, `output/bazel`. Copy that to your `/usr/local/bin` directory.

```shell
sudo cp output/bazel /usr/local/bin/bazel
```

To make sure it's working properly, run `bazel` on the command line and verify it prints help text. Note: this may take 15-30 seconds to run, so be patient!

```shell
bazel

Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  build               Builds the specified targets.
  canonicalize-flags  Canonicalizes a list of bazel options.
  clean               Removes output files and optionally stops the server.
  dump                Dumps the internal state of the bazel server process.
  fetch               Fetches external repositories that are prerequisites to the targets.
  help                Prints help for commands, or the index.
  info                Displays runtime info about the bazel server.
  mobile-install      Installs targets to mobile devices.
  query               Executes a dependency graph query.
  run                 Runs the specified target.
  shutdown            Stops the bazel server.
  test                Builds and runs the specified test targets.
  version             Prints version information for bazel.

Getting more help:
  bazel help <command>
                   Prints help and options for <command>.
  bazel help startup_options
                   Options for the JVM hosting bazel.
  bazel help target-syntax
                   Explains the syntax for specifying targets.
  bazel help info-keys
                   Displays a list of keys used by the info command.
```

Move out of the `bazel` directory, and we'll move onto the next step.

```shell
cd ..
```

## Compiling TensorFlow

Download your desired TensorFlow version from TensorFlow github and extract it in 'tensorflow' directory :

```shell
wget https://github.com/tensorflow/tensorflow/archive/v1.6.0.zip
unzip -d tensorflow v1.6.0.zip
cd tensorflow
```

Once in the directory, we have to write a nifty one-liner that is incredibly important. The next line goes through all files and changes references of 64-bit program implementations (which we don't have access to) to 32-bit implementations. Neat!

```shell
grep -Rl 'lib64' | xargs sed -i 's/lib64/lib/g'
```

Next, we need to delete a particular line in `tensorflow/core/platform/platform.h`. Open up the file in your favorite text editor:

```shell
sudo nano tensorflow/core/platform/platform.h
```

Now, scroll down toward the bottom and delete the following line containing `#define IS_MOBILE_PLATFORM` (around line 48):

```cpp
#elif defined(__arm__)
#define PLATFORM_POSIX
...
#define IS_MOBILE_PLATFORM   <----- DELETE THIS LINE
```

This keeps our Orange Pi device (which has an ARM CPU) from being recognized as a mobile device.

Now let's configure the build:

```shell
./configure

Please specify the location of python. [Default is /usr/bin/python]: /usr/bin/python
Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -march=native]: 
Do you wish to use jemalloc as the malloc implementation? [Y/n] Y
Do you wish to build TensorFlow with Google Cloud Platform support? [y/N] N
Do you wish to build TensorFlow with Hadoop File System support? [y/N] N
Do you wish to build TensorFlow with the XLA just-in-time compiler (experimental)? [y/N] N
Please input the desired Python library path to use. Default is [/usr/local/lib/python2.7/dist-packages]
Do you wish to build TensorFlow with OpenCL support? [y/N] N
Do you wish to build TensorFlow with CUDA support? [y/N] N
```

_Note: if you want to build for Python 3, specify `/usr/bin/python3` for Python's location and `/usr/local/lib/python3.4/dist-packages` for the Python library path._

Bazel will now attempt to clean. This takes a really long time (and often ends up erroring out anyway), so you can send some keyboard interrupts (CTRL-C) to skip this and save some time.

Now we can use it to build TensorFlow! **Warning: This takes a really, really long time. Several hours.**

```shell
bazel build -c opt --copt="-mfpu=neon-vfpv4" --copt="-funsafe-math-optimizations" --copt="-ftree-vectorize" --copt="-fomit-frame-pointer" --local_resources 512,1.0,1.0 --verbose_failures tensorflow/tools/pip_package:build_pip_package
```

## Cleaning Up

There's one last bit of house-cleaning we need to do before we're done: remove the USB drive that we've been using as swap.

First, turn off your drive as swap:

```shell
sudo swapoff /dev/XXX
```

Finally, remove the line you wrote in `/etc/fstab` referencing the device

```
sudo nano /etc/fstab
```

Then reboot your Orange Pi.

**And you're done!** You deserve a break.

## References

1. [tensorflow-on-raspberry-pi](https://github.com/samjabrahams/tensorflow-on-raspberry-pi)
2. [tensorflow-on-orange-pi](https://github.com/snowsquizy/tensorflow-on-orange-pi)

## Some Tips

soon
