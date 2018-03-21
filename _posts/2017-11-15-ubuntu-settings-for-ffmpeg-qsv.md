---
layout: post
title: "Ubuntu 16.04에서 FFMPEG QSV 빌드하기"
tags: [ubuntu, ffmpeg, qsv]
comments: true
share: true
---


### Media Server Studio (MSS) 설치하기

인텔에서 설치 가이드를 제공한다.

> <https://software.intel.com/en-us/articles/how-to-setup-media-server-studio-on-secondary-os-of-linux>

#### 준비

현재 user를 video group에 포함 (필수 아님, 권장 사항)
```bash
sudo usermod -a -G video $USER
```

다음 명령어로 장치가 보이는지 확인 했을때,
```bash
lspci -nn -s 00:02.0
```
이런 식으로 나와야 한다.

```
00:02.0 VGA compatible controller [0300]: Intel Corporation blah blah
```

ESXi 가상환경에서 Intel iGPU를 passthrough 한 guest에서 PCI bus id가 00:02.0로 나오는 것이 아니라 03: 으로 시작한다. 이게 명확한 문제인지는 모르겠지만 ESXi 가상 환경에서는 전부 실패했고 성공한 사례도 아직 본 적이 없다. (물론 linux 환경에서 말이다.) 인텔 [WHITE PAPER](https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/quicksync-video-ffmpeg-install-valid.pdf) page 11에 보면 valid PCI id를 가져야 한다고 명시하고 있는데 drm 문제로 정확한 pci id 통해 인텔 기기를 확인하는 듯 하다. 커널 빌드 시에 해당 사항을 패치에 포함 해주면 될 것 같은데 능력 밖이라...

#### MSS 다운로드

CPU와 플랫폼에 맞는 버전([참고](https://software.intel.com/en-us/articles/driver-support-matrix-for-media-sdk-and-opencl))을 다운로드한다. 리눅스 커널 모드 패치 (KMD)가 필수적인데, MSS 버전마다 제공하는 커널 패치가 다르다. 2016latest 버전에는 3.14.5, 2017R3에서는 4.4를 제공한다. 예를 들어, 4세대 하스웰은 2016 버전까지만 공식 지원하는데 Ubuntu 16.04 LTS에 2016latest를 깔고 커널 패치를 하자면 컴파일 에러가 많아 애로사항이 꽃핀다. 5세대 브로드웰 이상이라면 2017R3를 사용하자.

* [다운로드 홈](https://software.intel.com/en-us/intel-media-server-studio)
* [2017R3](http://registrationcenter-download.intel.com/akdlm/irc_nas/vcp/11800/MediaServerStudioEssentials2017R3.tar.gz)
* [2016 Latest](http://registrationcenter-download.intel.com/akdlm/irc_nas/vcp/8684/MediaServerStudioEssentials2016.tar.gz)

다운로드해서 압축을 풀어 준다.

```bash
wget http://registrationcenter-download.intel.com/akdlm/irc_nas/vcp/11800/MediaServerStudioEssentials2017R3.tar.gz
tar xf MediaServerStudioEssentials2017R3.tar.gz
cd MediaServerStudioEssentials2017R3
tar xf SDK2017Production16.5.2.tar.gz
cd SDK2017Production16.5.2/Generic/
tar xf intel-linux-media_generic_16.5.2-64009_64bit.tar.gz
```

#### 설치

```./install_media.sh```을 실행하여 설치하거나 (CentOS를 기본으로 하는데 Generic이라고 Ubuntu에서도 가능하다.) 아니면 인텔 가이드처럼 폴더를 복사해서 설치할 수 있다. ```SDK2017Production16.5.2/Generic/``` 아래의 내용이 복사 될 수 있도록 하자. 아래 쉘 스크립트 참조.

#### 리눅스 커널 패치 및 환경 변수 등록

대략적인 과정은 빌드 관련 패키지를 설치하고 리눅스 커널을 다운 받은 다음 MSS에 있는 패치를 적용해서 빌드 후 설치이다. 디스크 용량과 시간이 상당히 소요되니 감안하여 진행한다. 아래는 MSS 설치 전과정을 수행하는 스크립트이다. 관리자 권한이 필요하다.

```bash
#!/bin/bash
set -e

echo "remove other libdrm/libva"
find /usr -name "libdrm*" | xargs rm -rf
find /usr -name "libva*" | xargs rm -rf

echo "Remove old MSS install files ..."
rm -rf /opt/intel/mediasdk
rm -rf /opt/intel/common
rm -rf /opt/intel/opencl

echo "download MSS 2017R3"
wget http://registrationcenter-download.intel.com/akdlm/irc_nas/vcp/11800/MediaServerStudioEssentials2017R3.tar.gz
tar xf MediaServerStudioEssentials2017R3.tar.gz
cd MediaServerStudioEssentials2017R3
tar xf SDK2017Production16.5.2.tar.gz
cd SDK2017Production16.5.2/Generic/
tar xf intel-linux-media_generic_16.5.2-64009_64bit.tar.gz

echo "install user mode components"
#put the generic components in standard locations
/bin/cp -r etc/* /etc
/bin/cp -r lib/* /lib
/bin/cp -r opt/* /opt
/bin/cp -r usr/* /usr

#ensure that new libraries can be found
echo '/usr/lib64' > /etc/ld.so.conf.d/libdrm_intel.conf
echo '/usr/local/lib' >> /etc/ld.so.conf.d/libdrm_intel.conf
ldconfig

echo "install kernel build dependencies"
apt-get -y install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc g++

echo "download 4.4 kernel"
if [ ! -f ./linux-4.4.tar.xz ]; then
     wget https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.4.tar.xz
fi
tar -xJf linux-4.4.tar.xz

echo "apply kernel patches"
cp  /opt/intel/mediasdk/opensource/patches/kmd/4.4/intel-kernel-patches.tar.bz2 .
tar -xvjf intel-kernel-patches.tar.bz2
cd linux-4.4
for i in ../intel-kernel-patches/*.patch; do patch -p1 < $i; done

echo "build patched 4.4 kernel"
make olddefconfig
make -j 8
make modules_install
make install

echo "append eivornment variables"
echo 'LD_LIBRARY_PATH="/usr/local/lib:/usr/lib64"' >> /etc/environment
echo 'LIBVA_DRIVER_NAME=iHD' >> /etc/environment
echo 'LIBVA_DRIVERS_PATH=/opt/intel/mediasdk/lib64' >> /etc/environment
```

#### 재부팅

그냥 재부팅하면 안되고 부팅할때 shift를 눌러서 패치된 커널로 부팅하도록 해야한다. Grub을 수정해서 원하는 커널로 부팅하도록 할 수 있다. [참고](https://unix.stackexchange.com/questions/198003/set-default-kernel-in-grub)

#### 설치 확인하기

1\. ```vainfo```를 실행해서 보면 된다.

```bash
$ vainfo
libva info: VA-API version 0.99.0
libva info: va_getDriverName() returns 0
libva info: User requested driver 'iHD'
libva info: Trying to open /opt/intel/mediasdk/lib64/iHD_drv_video.so
libva info: Found init function __vaDriverInit_0_32
libva info: va_openDriver() returns 0
vainfo: VA-API version: 0.99 (libva 1.67.0.pre1)
vainfo: Driver version: 16.5.2.64009-ubit
vainfo: Supported profile and entrypoints
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264ConstrainedBaseline: <unknown entrypoint>
      VAProfileH264ConstrainedBaseline: <unknown entrypoint>
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264Main               : <unknown entrypoint>
      VAProfileH264Main               : <unknown entrypoint>
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      VAProfileH264High               : <unknown entrypoint>
      VAProfileH264High               : <unknown entrypoint>
      VAProfileMPEG2Simple            : VAEntrypointEncSlice
      VAProfileMPEG2Simple            : VAEntrypointVLD
```

User requested driver가 i965가 아니라 iHD로 잡혀야 한다.

```
vainfo: error while loading shared libraries: libXfixes.so.3: cannot open shared objec
t file: No such file or directory
```
이런 에러가 나온다면 아래 dependency를 설치해준다.
```
sudo apt-get -y install libxfixes-dev libpicaccess0
```

2\. ```uname -r```로 커널도 확인한다.

3\. 인텔에서 환경을 체크하는 툴도 제공한다. [링크](https://software.intel.com/en-us/articles/mss-sys-analyzer-linux)

4\. MSS에 포함된 샘플 코드를 돌려본다.
```bash
cd /opt/intel/mediasdk/samples/_bin/x64
./sample_multi_transcode -i::h264 ../content/test_stream.264 -o::h264 out.h264 -hw -la
```

가상 환경에서 돌리면 이런식으로 나오는데,
```bash
[ERROR], sts=MFX_ERR_DEVICE_FAILED(-17), main, transcode.Init failed at /home/lab_msdk/buildAgentDir/buildAgent_MediaSDK1/git/mdp_msdk-lib/samples/sample_multi_transcode/src/sample_multi_transcode.cpp:750
```

[여기](https://github.com/Intel-Media-SDK/MediaSDK/blob/master/api/include/mfxdefs.h)에 나온 바로는 역시 장치 연결이 잘 되지 않아서 그런것 같다.


#### 자체 빌드하기 (선택사항)

필요한 파일과 라이브러리는 제공되는 것을 써도 되지만 직접 빌드해서 설치해도 된다. [Brainiarc7의 gist 문서](https://gist.github.com/Brainiarc7/dd9e9b62bddb53b771b3a0754e352f53)와 [인텔 가이드](https://software.intel.com/en-us/articles/how-to-setup-media-server-studio-on-secondary-os-of-linux) 마지막을 보면 된다. libva 빌드를 위해서는 libdrm이 필요하므로 libdrm을 꼭 먼저 빌드하라고 한다.







### FFMPEG 빌드하기

참고 문서
> <https://software.intel.com/en-us/articles/accessing-intel-media-server-studio-for-linux-codecs-with-ffmpeg>

QSV를 지원하는 FFMPEG는 두 가지 빌드 옵션이 필수이다. ```--enable-libmfx --enable-nonfree```

#### Dependency 설치

```bash
sudo apt-get -y install gcc gobjc pkg-config libpthread-stubs0-dev libpciaccess-dev make patch yasm g++ autoconf cmake automake build-essential libass-dev libfreetype6-dev libgpac-dev libsdl1.2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libx11-dev libxext-dev libxfixes-dev texi2html zlib1g-dev libx264-dev libmp3lame-dev libfaac-dev librtmp-dev libvo-aacenc-dev libx264-dev cifs-utils libbz2-dev
```

#### libmfx 준비

header 파일 준비

```
# mkdir /opt/intel/mediasdk/include/mfx
# cp /opt/intel/mediasdk/include/*.h /opt/intel/mediasdk/include/mfx
```

pkg-config를 정의 해준다.

```sudo vi /usr/lib64/pkgconfig/libmfx.pc```:

```bash
prefix=/opt/intel/mediasdk
exec_prefix=${prefix}
libdir=${prefix}/lib/lin_x64
includedir=${prefix}/include

Name: libmfx
Description: Intel Media SDK 2017R3
Version: 16.5.2
Libs: -L${libdir} -lmfx -lva -lva-drm -lstdc++ -ldl
Cflags: -I${includedir} -I/usr/include/libdrm
```

컴파일러가 찾을 수 있도록...

```bash
export PKG_CONFIG_PATH=/usr/lib64/pkgconfig:/usr/lib/pkgconfig
```

#### 컴파일

Deploy ffmpeg to `/opt/ffmpeg_qsv/`

```bash
wget http://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2
tar xf ffmpeg-snapshot.tar.bz2
cd ffmpeg
./configure --disable-shared --enable-static --enable-gpl --enable-nonfree --enable-libmfx --enable-fontconfig --enable-libfreetype --enable-libmp3lame --enable-librtmp --enable-libx264 --enable-version3 --enable-ffplay --disable-doc --disable-ffserver --enable-pthreads --enable-filters --enable-libvorbis --enable-libtheora --enable-runtime-cpudetect --enable-libass --enable-bzlib --enable-zlib --prefix=/opt/ffmpeg_qsv/ --enable-vaapi
make -j$(nproc)
make install
```

#### 인코딩 테스트

점유율 확인은 아래 툴로 하면 된다.

```bash
sudo apt-get -y install intel-gpu-tools htop
sudo intel_gpu_top
```

1\. cpu only

```bash
/opt/ffmpeg_qsv/bin/ffmpeg -i in.mp4 -vcodec libx264 -an out_libx264.mp4
```

![]({{ site.url }}{{ site.baseurl }}/img/PicPick_Capture_20171110_001.png)

CPU를 아주 효율적으로 쓰고 (갈구고) 속도는 약 1배속


2\. QSV

```bash
/opt/ffmpeg_qsv/bin/ffmpeg -i in.mp4 -vcodec h264_qsv -init_hw_device qsv:hw -an out_264qsv.mp4
```

![]({{ site.url }}{{ site.baseurl }}/img/PicPick_Capture_20171113_001.png)

속도는 x7.5인데 이때도 CPU를 생각보다 많이 쓴다.


3\. vaapi

참고로 vaapi도 유사하게 빌드하면 되고 ffmpeg 명령어는 다음과 같다.

```bash
/opt/ffmpeg_vaapi/bin/ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i in.mp4 -an -vf 'format=nv12,hwupload' -c:v h264_vaapi out_264vaapi.mp4
```


### 참고 자료

* [Generic Linux* Intel® Media Server Studio Installation](https://software.intel.com/en-us/articles/how-to-setup-media-server-studio-on-secondary-os-of-linux)
* [Accessing Intel® Media Server Studio for Linux* codecs with FFmpeg](https://software.intel.com/en-us/articles/accessing-intel-media-server-studio-for-linux-codecs-with-ffmpeg)
* <https://gist.github.com/Brainiarc7/dd9e9b62bddb53b771b3a0754e352f53>
* <http://blog.djjproject.com/253>
* <https://ubuntuforums.org/showthread.php?t=2329355>
* <http://blog.guorongfei.com/2015/10/08/intel_qsv_linux/>
* <http://blog.csdn.net/LeoChen1983/article/details/72742656>
* <https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/quicksync-video-ffmpeg-install-valid.pdf>
