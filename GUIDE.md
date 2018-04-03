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

Now you can compile bazel with the command below. This takes a lot of time and in my case it took around half a day.

```shell
sudo ./compile.sh
```

When the build finishes, you end up with a new binary, `output/bazel`. Copy that to your `/usr/local/bin` directory.

```shell
sudo cp output/bazel /usr/local/bin/bazel
```

To make sure it's working properly, run `bazel` on the command line and verify it prints help text. Note: this may take 15-30 seconds to run, so be patient!

```
bazel

[bazel release 0.11.1- (@non-git)]
Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  build               Builds the specified targets.
  canonicalize-flags  Canonicalizes a list of bazel options.
  clean               Removes output files and optionally stops the server.
  coverage            Generates code coverage report for specified test targets.
  cquery              Loads, analyzes, and queries the specified targets w/ configurations.
  dump                Dumps the internal state of the bazel server process.
  fetch               Fetches external repositories that are prerequisites to the targets.
  help                Prints help for commands, or the index.
  info                Displays runtime info about the bazel server.
  license             Prints the license of this software.
  mobile-install      Installs targets to mobile devices.
  print_action        Prints the command line args for compiling a file.
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
Now let's configure the build, enter options as entered below :

```
./configure
You have bazel 0.11.1- (@non-git) installed.
Please specify the location of python. [Default is /usr/bin/python]: /usr/bin/python3

Found possible Python library paths:
  /usr/local/lib/python3.5/dist-packages
  /usr/lib/python3/dist-packages
Please input the desired Python library path to use.  Default is [/usr/local/lib/python3.5/dist-packages]

Do you wish to build TensorFlow with jemalloc as malloc support? [Y/n]: y
jemalloc as malloc support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Google Cloud Platform support? [Y/n]: n
No Google Cloud Platform support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Hadoop File System support? [Y/n]: n
No Hadoop File System support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Amazon S3 File System support? [Y/n]: n
No Amazon S3 File System support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Apache Kafka Platform support? [y/N]: n
No Apache Kafka Platform support will be enabled for TensorFlow.

Do you wish to build TensorFlow with XLA JIT support? [y/N]: n
No XLA JIT support will be enabled for TensorFlow.

Do you wish to build TensorFlow with GDR support? [y/N]: n
No GDR support will be enabled for TensorFlow.

Do you wish to build TensorFlow with VERBS support? [y/N]: n
No VERBS support will be enabled for TensorFlow.

Do you wish to build TensorFlow with OpenCL SYCL support? [y/N]: n
No OpenCL SYCL support will be enabled for TensorFlow.

Do you wish to build TensorFlow with CUDA support? [y/N]: n
No CUDA support will be enabled for TensorFlow.

Do you wish to build TensorFlow with MPI support? [y/N]: n
No MPI support will be enabled for TensorFlow.

Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -march=native]: 

Would you like to interactively configure ./WORKSPACE for Android builds? [y/N]: n
Not configuring the WORKSPACE for Android builds.

Preconfigured Bazel build configs. You can use any of the below by adding "--config=<>" to your build command. See tools/bazel.rc for more details.
        --config=mkl            # Build with MKL support.
        --config=monolithic     # Config for mostly static monolithic build.
        --config=tensorrt       # Build with TensorRT support.
Configuration finished
```
Before building tensor one important step needs to be taken otherwise the resulted python wheel of TensorFlow will give you error when you try to import it inside python and that means you have wasted all of your precious time ! What we are going to do is commenting out some lines according to this [issue](https://github.com/tensorflow/tensorflow/issues/17986) :

```shell
sed -i -e 's/#include "tensorflow\/core\/kernels\/concat_lib.h"/\/\/#include "tensorflow\/core\/kernels\/concat_lib.h"/g' core/kernels/list_kernels.h
sed -i -e 's/#include "tensorflow\/core\/kernels\/concat_lib.h"/\/\/#include "tensorflow\/core\/kernels\/concat_lib.h"/g' core/kernels/list_kernels.cc
sed -i -e 's/ConcatCPU<T>(c->device(), inputs_flat, &output_flat);/\/\/ConcatCPU<T>(c->device(), inputs_flat, &output_flat);/g' core/kernels/list_kernels.h
```
Now we are ready to build TensorFlow which on Orange Pi Zero is a little more time-consuming than 1GB RAM boards (when I say a little more it means maybe 3 to 10 times more !) so fire the command below and just leave your OPZ doing it's job while you check on it randomly.

```shell
bazel build -c opt --copt="-mfpu=neon-vfpv4" --copt="-funsafe-math-optimizations" --copt="-ftree-vectorize" --copt="-fomit-frame-pointer" --local_resources 512,1.0,1.0 --verbose_failures tensorflow/tools/pip_package:build_pip_package
```

During build process, your Orange Pi Zero may freeze completely and you may not be able to reach it through ssh so it doesn't necessary mean that it is actually compiling anything and many times it means that you need to restart your device and run the above command again to continue the build process. I have provided some times in [tips](#some-tips) section to avoid these situations as much as possible. And finally you will see this line :

```shell
INFO: Build completed successfully, 4 total actions
```
Now you can finally make a python wheel, issue the following command :

```shell
bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```
This one will not take too much time and you can install your python package form `/tmp/tensorflow_pkg` directory :
```shell
pip3 install --user /tmp/tensorflow_pkg/ensorflow-1.6.0-cp35-cp35m-linux_armv7l.whl
```
And your TensorFlow is ready to use.

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

## References

1. [tensorflow-on-raspberry-pi](https://github.com/samjabrahams/tensorflow-on-raspberry-pi)
2. [tensorflow-on-orange-pi](https://github.com/snowsquizy/tensorflow-on-orange-pi)

## Some Tips

Here are some tips to build TensorFlow and Bazel as fast as possible while preventing from occasional freezes that can happen on Orange Pi Zero, these are my own experiences and may or may not work for you :

* Use at least a 4GB flash memory drive (I saw some peaks of more than 1GB swap usage during my experience).
* Use a USB3 flash memory drive (While OPZ has only USB2 port but because USB3 flash memories are generally faster even on USB2 ports they can speed up the process).
* Disable any swap on SD card (Armbian comes with some swap space on SD card itself which slows down the building procedure, you can disable it from /etc/fstab)
