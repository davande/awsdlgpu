# Setup An Amazon EC2 GPU Instance for Deep Learning

__Last updated: 2014-12-11__

## From scratch

### Create EC2 GPU Instance

After you registered at AWS (Amazon Web Services), you should be able to create EC2 instance.

Choose `Launch`, then select `Ubuntu Server 14.04 LTS (HVM), SSD Volume Type`, then choose `g2.2xlarge` instance type.

The initial storage on the instance settings is 8GB, but it's far from enough. I personally pushed to 200GB so that I can have enough room for installing packages, drivers, databases, etc.

### Install packages.

You got a brand new Ubuntu machine after your instance is running successfully.

Home folder of the instance is empty, for further usage, we can create a `Downloads` folder to store all downloads

```
$ mkdir Downloads
```

Firstly, you need to update your update library by:

```
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get install linux-headers-generic linux-headers-virtual linux-image-virtual linux-virtual
$ sudo apt-get install linux-image-extra-virtual
```

__Reboot your instance by stopping and starting it.__

Then, you need to install `build-essential` to enable your development support:

```
$ sudo apt-get install build-essential binutils
```

Some supporting libraries needed for getting and building your project later:

```
$ sudo apt-get install git cmake
```

Theano and Caffe are two popular deep learning framework. In this instance, we are going to support them. You need to install following packages:

```
$ sudo apt-get install libopenblas-dev libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libboost-all-dev libhdf5-serial-dev libgflags-dev libgoogle-glog-dev liblmdb-dev protobuf-compiler
```

Your BLAS in this case is openBLAS.

You may also need Java support for some cases:

```
$ sudo apt-get install openjdk-7-jdk
```

__Reboot your instance by stopping and starting it.__

### Scientific Python Support from Anaconda

Anaconda is an awesome Python distribution for large-scale data processing, predictive analysis, and scientific computing. It contains many well-known packages and maintained it well. Therefore, instead of messing with Ubuntu's Python, we use Anaconda's Python. Anaconda provides a clean installation, and once you don't need it, you can simply delete it from home folder.

First, get Anaconda (use Anaconda Python 2.7) (Note that as I'm writing, latest Anaconda is 2.1.0, you can download the latest version from [here](http://continuum.io/downloads) also):

```
$ cd Downloads
$ wget http://09c8d0b2229f813c1b93-c95ac804525aac4b6dba79b00b39d1d3.r79.cf1.rackcdn.com/Anaconda-2.1.0-Linux-x86_64.sh
```

After download, you simply run the bash file:

```
$ bash ./Anaconda-2.1.0-Linux-x86_64.sh
```

Follow the default instruction, you should be able to find a new folder: `anaconda` in your home folder. Now if you check you Python's version, it should give you:

```
Python 2.7.8 :: Anaconda 2.1.0 (64-bit)
```

Otherwise, you need to add Anaconda's path to your `.bashrc`

```
$ nano ~/.bashrc
```
Add following line to the end of the file

```
export PATH="/home/ubuntu/anaconda/bin:$PATH"
```
Press `Ctrl+X` to save and exit, then you need to source the file to enable current setting:

```
$ source ~/.bashrc
```

### GPU Driver and CUDA Support

Check your instance's graphic card by:

```
$  lspci | grep "VGA"
```
It should give you something similar to this:

```
00:02.0 VGA compatible controller: Cirrus Logic GD 5446
00:03.0 VGA compatible controller: NVIDIA Corporation GK104GL [GRID K520] (rev a1)
```

Nvidia Grid K520 is a Cloud Gaming Graphic Card, it got 2 GK104 GPUs where each of GPU has 1536 CUDA cores. The compute capability is 3.0, so you can install latest cuDNN support.

You need to download the driver for Grid K520 firstly from [here](http://www.nvidia.com/Download/index.aspx?lang=en-us). You also can use this address to download:

```
$ wget http://us.download.nvidia.com/XFree86/Linux-x86_64/340.65/NVIDIA-Linux-x86_64-340.65.run
```

Try to install the driver now by

```
sudo bash NVIDIA-Linux-x86_64-340.65.run
```  
You will not be able to install it because `nouveau` of the system is still on. The installer will add a blacklist to `nouveau` and quit. After the installer quited, you need to update your system by:

```
$ sudo update-initramfs -u
```

__Reboot your instance by stopping and starting it.__

Then you can simply install the driver by:

```
$ sudo bash NVIDIA-Linux-x86_64-340.65.run
```

You need to get recent CUDA Toolkit in order to use your GPU:

```
$ wget http://developer.download.nvidia.com/compute/cuda/6_5/rel/installers/cuda_6.5.14_linux_64.run
```

And install CUDA by:
```
$ sudo bash cuda_6.5.14_linux64.run
```

**DO NOT INSTALL GPU DRIVER INSIDE THE CUDA TOOLKIT**

Add following line to end of your `.bashrc` file.

```
# This configuration uses CUDA's symbolic link
export PATH=$PATH:/usr/local/cuda/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64 #if 64bit machine
export CUDA_ROOT=/usr/local/cuda
```

### Theano support

Create Theano's configuration file:

```
$ touch $HOME/.theanorc
```
and modify it as

```
[global]
floatX = float32
device = gpu0

[nvcc]
fastmath = True

[cuda]
root = /usr/local/cuda
```

You can use [this test](http://deeplearning.net/software/theano/tutorial/using_gpu.html#testing-theano-with-gpu) to valid your installation.

### Caffe support [UNDER REVISION]

Caffe recently supports Nvidia's new machine learning library --- cuDNN. It improves Caffe's performance. Note that cuDNN requires that the graphic card has at least 3.0 of compute capability. Graphic card of this instance is 3.0.

Download cuDNN from:

```
$ wget http://arl.fsktm.um.edu.my/cudnn-6.5-linux-R1.tgz
```
Extract cuDNN

```
$ tar zxvf cudnn-6.5-linux-R1.tgz
```
Copy extracted files to CUDA folder

```
$ sudo cp cudnn.h /usr/local/cuda-6.5/include
$ sudo cp libcudnn* /usr/local/cuda-6.5/lib64
```

Clone Caffe from GitHub

```
$ cd ~
$ git clone https://github.com/BVLC/caffe
```

There are several libraries needed for Caffe's Python support, we can install them by:

```
$ pip install leveldb
$ pip install protobuf
$ pip install python-gflags
```

After you installed Python dependencies, we can start to modify Caffe's `make` configurations.

1. Enable `USE_CUDNN := 1`
2. Change `BLAS := atlas` to `BLAS := open`
3. Comment system's Python support and enable Anaconda's Python support.
4. Comment system's Python library support and enable Anaconda's Python library support.

Some Anaconda release has bad `libm` library, we need to force the compiler to choose the one from system:

```
$ cd ~/anaconda/lib
$ mv libm.so.6 libm.so.6.tmp
$ mv lib.so lib.so.tmp
```

Your final `.bashrc` should look like this

```
# added by Anaconda 2.1.0 installer
export PATH="/home/ubuntu/anaconda/bin:$PATH"
export PATH=$PATH:/usr/local/cuda/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64:/home/ubuntu/anaconda/lib:/usr/local/lib:/usr/lib
export CUDA_ROOT=/usr/local/cuda
export CAFFE_ROOT=/home/ubuntu/caffe
```

Build Caffe

```
$ make all -j8
$ make test
$ make runtest
```

You should have 2 Disabled tests, the reason is from OpenCV part.

## From AMI (Amazon Machine Image)

I made a ready-to-use AMI so that you don't have to be so painful for these technical details.

I made this AMI public so that you can use it to launch a new instance or make a spot request.

Search **DGYDLGPUv3** (ami-c5c2ee97) from public AMIs.

## Contacts

Hu Yuhuang  
Advanced Robotic Lab  
Department of Artificial Intelligence  
Faculty of Computer Science & IT  
University of Malaya  
Email: duguyue100@gmail.com
