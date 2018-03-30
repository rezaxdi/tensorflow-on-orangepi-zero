
# Installing TensorFlow on Orange Pi Zero (Compiled and tested on 512MB version)
This repo provides 2 ways to install TensorFlow on Orange Pi Zero, the first and easy one is using a pre compiled python wheel and the second one is by compiling it yourself if the first one fails or if the wheel doesn't suit your needs. This repo can be considered as a fork of [this](https://github.com/samjabrahams/tensorflow-on-raspberry-pi) one but as I wanted to have different release files for Orange Pi only and keep it as clean as possible created it from scratch.
## Index 
1. Installing from wheel
2. Installing from source
3. Credits
4. License

## Installing from wheel 
**Note: These are unofficial binaries (though built from the minimally modified official source), and thus there is no expectation of support from the TensorFlow team. Please don't create issues for these files in the official TensorFlow repository.**

Pre-built binary is built using Armbian 5.41 and is targeted for Orange Pi Zero running Armbian 5.41, so this method may or may not work for you. I currently only make wheels for latest versions of python 3.x available on Armbian. 
* Install Dependencies : `sudo apt update & sudo apt install python3-pip python3-dev`
* Download latest release : `wget https://github.com/rezaxdi/tensorflow-on-orangepi-zero/releases/download/v1.6.0/tensorflow-1.6.0-cp35-cp35m-linux_armv7l.whl`
* Install the wheel : `pip3 install --user tensorflow-1.6.0-cp35-cp35m-linux_armv7l.whl`

## Installing from source
If the earlier method didn't work for you then you can build TensorFlow from the source. Orange Pi Zero suffers from lack of the memory and this can lead to occasional freezes during the build process even when you use lots of swap space so this is a process that can take more than 24 hour and even several days in the worst case. If you are ready follow the guide carefully, in the last part of the guide, I have provided some useful tips to prevent those occasional freezes as much as possible :

[Building from source guide](https://github.com/rezaxdi/tensorflow-on-orangepi-zero)

## Credits 

As told in the beginning this repo can be considered as a fork of @samjabrahams work so I will bring his repo credit section in continue but before that I want to add some more, it took a lot of time to compile binaries on Orange Pi Zero and after around 8 days [this](https://github.com/tensorflow/tensorflow/issues/17986) issue was just a lot of disappointment so thanks from @Lexicographical which finally made my binaries working in Orange Pi Zero.

>While the final pieces of grunt work were done primarily by @samjabrahams and @petewarden, this effort has been going on for almost as long as TensorFlow has been open-source, and involves work that spans multiple months in separate codebases. This is an incomprehensive list of people and their work @samjabrahams ran across while working on this.

>The majority of the source-building guide is a modified version of [these instructions for compiling TensorFlow on a Jetson TK1](http://cudamusing.blogspot.com/2015/11/building-tensorflow-for-jetson-tk1.html). Massimiliano, you are the real MVP. Note: the TK1 guide was [updated on June 17, 2016](http://cudamusing.blogspot.com/2016/06/tensorflow-08-on-jetson-tk1.html)

>@vmayoral put a huge amount of time and effort trying to put together the pieces to build TensorFlow, and was the first to get something close to a working binary.

>A bunch of awesome Googlers working in both the TensorFlow and Bazel repositories helped make this possible. In no particular order: @vrv, @damienmg, @petewarden, @danbri, @ulfjack, @girving, and @nlothian
    
## License

The file RELEASE_LICENSE is TensorFlow's license file and applies to the binaries distributed in the [releases](https://github.com/rezaxdi/tensorflow-on-orangepi-zero/releases).

The file LICENSE is for the repository itself.
