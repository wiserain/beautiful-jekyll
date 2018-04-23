---
layout: post
title: "우분투 16.04 딥러닝 라이브러리 기본 환경"
tags: [ubuntu, 16.04, nvidia, cuda, cudnn]
comments: true
share: false
---

### check gpu
```bash
sudo update-pciids
lspci -nn | grep '\[03'
```

### nvidia driver, cuda, cuDNN
> https://gist.github.com/morgangiraud/990cf65dcb27068a4ca6b9db4957acc7
> https://yunsangq.github.io/articles/2017-02/Ubuntu-16.04(64bit),-CUDA-8.0,-cuDNN-5.1-Install

```bash
sudo apt-get update
sudo apt-get upgrade

# add ppa graphic driver repository
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt-get update

# check which version is available
apt list nvidia-*

sudo apt install nvidia-396

# check driver installed
sudo shutdown -r now
watch -n1 nvidia-smi

# If it doesn't work, sometimes this is due to a secure boot option of your motherboard, disable it and test again

# download and install cuda deb files
sudo dpkg -i cuda-repo-ubuntu1604-9-0*
sudo apt update && sudo apt install cuda

# Add cuda to your PATH and install the toolkit
# Also add them to your .bashrc file
export PATH=/usr/local/cuda-9.0/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-9.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
export CUDA_HOME=/usr/local/cuda-9.0

source ~/.bashrc
nvcc --version

# download and install cudnn deb files
sudo dpkg -i libcudnn7*+cuda9.0_amd64.deb
```

### basic python packages for deep learning
```bash
sudo apt install python-pydot python-pydot-ng graphviz \
  python-matplotlib python-imaging -y
```

### troubleshooting
1\. ```_tkinter.TclError: no display name and no $DISPLAY environment variable```
> https://github.com/enigmampc/catalyst/issues/39

```echo "backend: Agg" > ~/.config/matplotlib/matplotlibrc```

### keras-contrib

```bash
sudo pip install git+https://www.github.com/keras-team/keras-contrib.git
```
