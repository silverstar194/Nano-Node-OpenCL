# Nano-Node-OpenCL
Implementation for Nano Node with OpenCL on AWS p2.xlarge Ubuntu 14.02

### Overview
Below I outline the steps I took to run Nano Node 11.2 on AWS p2.xlarge Ubuntu 14.02. The node utilizes OpenCL to support the Tesla K80 GPU. The GPU benchmarked at under a second each for PoW. I also give outlines of how to enable remote access to the node.

### Install Nvidia Drives and Open CL
First make sure system is up to date:
```
sudo apt-get update
sudo apt-get upgrade
```

Next make sure you have all necessary dependences:
```
sudo apt-get install -y build-essential git python-pip libfreetype6-dev libxft-dev libncurses-dev libopenblas-dev gfortran python-matplotlib libblas-dev liblapack-dev libatlas-base-dev python-dev python-pydot linux-headers-generic linux-image-extra-virtual unzip python-numpy swig python-pandas python-sklearn unzip wget pkg-config zip g++ zlib1g-dev libcurl3-dev
sudo pip install -U pip
```

#### Nvidia Drives
Download the CUDA drives neccessary:
```
wget https://developer.nvidia.com/compute/cuda/8.0/prod/local_installers/cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64-deb
sudo dpkg -i cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64-deb
rm cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64-deb
sudo apt-get update
sudo apt-get install -y cuda
```

Make sure to export all paths needed for drives:
```
export CUDA_HOME=/usr/local/cuda
export CUDA_ROOT=/usr/local/cuda
export PATH=$PATH:$CUDA_ROOT/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$CUDA_ROOT/lib64
```

At this point you should be able to access the GPU and see information about it. Let's test it out:
```
nvidia-smi
```

Output:
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.111                Driver Version: 384.111                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla K80           Off  | 00000000:00:1E.0 Off |                    0 |
| N/A   41C    P0    81W / 149W |      0MiB / 11439MiB |     83%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

Next check the verison:
```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2013 NVIDIA Corporation
Built on Wed_Jul_17_18:36:13_PDT_2013
Cuda compilation tools, release 5.5, V5.5.0
```


#### OpenCL
Is that all looks good its time to move on to OpenCL:
```
sudo apt-get install ocl-icd-opencl-dev
sudo apt-get install nvidia-cuda-toolkit
sudo apt-get install clinfo
```

Now test that the installation when as exspected:
```
 clinfo
```

Output:
```Number of platforms:				 1
  Platform Profile:				 FULL_PROFILE
  Platform Version:				 OpenCL 1.2 CUDA 9.0.282
  Platform Name:				 NVIDIA CUDA
  Platform Vendor:				 NVIDIA Corporation
  Platform Extensions:				 cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_fp64 cl_khr_byte_addressable_store cl_khr_icd cl_khr_gl_sharing cl_nv_compiler_options cl_nv_device_attribute_query cl_nv_pragma_unroll cl_nv_copy_opts cl_nv_create_buffer
  
  .......
  
 ```

### Installing Nano Node Natively

At this point your system is ready to setup the Nano Node itself. I at first used the docker image. This will not work. The image as no way to utilize the GPU and will only use the CPU. I benchmarked the PoW at ~10 secounds on p2.xlarge uisng solely CPUs.

In order to use the GPU you MUST compile natively.

#### Dependency Build Instructions
Note: I had to update gcc on Ubuntu 14.02
```
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get install gcc-5 g++-5
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 60 --slave /usr/bin/g++ g++ /usr/bin/g++-5
```

```
sudo apt-get update && sudo apt-get upgrade   
sudo apt-get install git cmake g++ curl wget
```

Build Boost:
```
wget -O boost_1_66_0.tar.gz https://netix.dl.sourceforge.net/project/boost/boost/1.66.0/boost_1_66_0.tar.gz   
tar xzvf boost_1_66_0.tar.gz   
cd boost_1_66_0   
./bootstrap.sh --with-libraries=filesystem,iostreams,log,program_options,thread   
./b2 --prefix=../[boost] link=static install   
cd ..
```

Finally get the Node:
```
git clone --recursive https://github.com/nanocurrency/raiblocks.git rai_build   
cd rai_build   
cmake -DBOOST_ROOT=../[boost]/ -G "Unix Makefiles"   
make rai_node   
cp rai_node ../rai_node && cd ..
./rai_node --diagnostics
```

Now you must enable OpenCL in RaiBlocks/config.json:
```
"opencl_enable": "false" --> "opencl_enable": "true"
```
You can leave the platform, devices and threads as-is.

Output:
```
Testing hash function
Testing key derivation function
Dumping OpenCL information
OpenCL found 1 platforms and 1 devices
Platform: 0
FULL_PROFILE
OpenCL 1.2 CUDA 9.0.282
NVIDIA CUDA
NVIDIA Corporation
cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_fp64 cl_khr_byte_addressable_store cl_khr_icd cl_khr_gl_sharing cl_nv_compiler_options cl_nv_device_attribute_query cl_nv_pragma_unroll cl_nv_copy_opts cl_nv_create_buffer
Device: 0
	Tesla K80
	NVIDIA Corporation
	FULL_PROFILE
	GPU
	Compiler available: true
	Compute units available: 13
```
As you see above the GPU is detected!

#### Enabling Remote Access
Now I wanted to be able to access the node outside of localhost. Note this can be danergerous as people can spam your node. Strongly consider IP filtering.

It turns out simply opening the port does not work. As of 04/16/2016 AWS instances by default do not ave IPv6 addresses only IPv4. The node runs on IPv6. Using socat you can forward traffic from IPv4 to IPv4.

Install socat:
```
sudo apt-get install socat
```

Forword traffic:
```
sudo socat TCP4-LISTEN:7076,fork,su=nobody TCP6:[::1]:7076
```

You should now be able to make calls to your node through the public IP. If you can't be sure to check:
1. Security Groups within AWS and IP/port filters
2. You have enabled RCP in RaiBlocks/config.json if you want to use RCP calls


From this point you need to wait for the node to fully sync or using the fast sync method. For fast sync see https://discordapp.com/channels/370266023905198083/370285680691249162?jump=405924348990324736

Feel free to reach out of Discord with questions @silverstar194

Outside References: 
http://expressionflow.com/2016/10/09/installing-tensorflow-on-an-aws-ec2-p2-gpu-instance/
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-nvidia-driver.html
https://github.com/nanocurrency/raiblocks/wiki/Build-rai_node-samples

Speical Thanks:
renesq_the_bananomonster and other Discord users who answered many questions

