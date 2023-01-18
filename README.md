# CrossCompileOpenCV4PI
A guide on how to cross compile OpenCV 4.7.0 for Rasperri Pi.
It is important to note, that the host system needs to be x86-64.

If your host system is a windows, follow the steps in Prerequisites. If your host system is a x86-64 Debian, you can skip the Prerequisites step of course.

## Prerequisites
First, download debian as a docker image
```
docker pull debian
```
Now start the newly downloaded docker image
```
docker run -it debian
```

## Preparation
First, make sure the container is updated:
```
apt update
apt upgrade
```
Next, let’s enable the armhf architecture on the x86-64 machine:
```
dpkg --add-architecture armhf
apt update
apt install qemu-user-static
```
Install Python 3 and libraries on debian system
```
apt-get install python3-dev
apt-get install python3-numpy
apt-get install libpython3-dev:armhf
```
Install Python 2 and libraries on debian system
```
apt-get install python-dev
apt install curl
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py
python2 get-pip.py
python2 -m pip install numpy
apt-get install libpython2-dev:armhf
```
If you need support for GUI programs, install these packages. If not, you can ignore the following command
```
apt install libgtk-3-dev:armhf libcanberra-gtk3-dev:armhf
```
Now we need to install other libraries, that are required for OpenCV
```
apt install libtiff-dev:armhf zlib1g-dev:armhf
apt install libjpeg-dev:armhf libpng-dev:armhf
apt install libavcodec-dev:armhf libavformat-dev:armhf libswscale-dev:armhf libv4l-dev:armhf
apt-get install libxvidcore-dev:armhf libx264-dev:armhf
```
Now we need to install the cross compilers from Debian which can be used to create armhf binaries for Raspberry Pi
```
apt install crossbuild-essential-armhf
apt install gfortran-arm-linux-gnueabihf
```
Now, lets download some libraries, that we need now:
```
apt install cmake git pkg-config wget
```
Next, we can download the current release of OpenCV (4.7.0 as of this writing). This example will contain the full installation of OpenCV (default and contrib libraries). If you only need default, leave the part, where I am downloading contrib and remove the OPENCV_EXTRA_MODULES_PATH parameter from the cmake command below.
```
cd ~
mkdir opencv_all && cd opencv_all
wget -O opencv.tar.gz https://github.com/opencv/opencv/archive/4.7.0.tar.gz
tar xf opencv.tar.gz
wget -O opencv_contrib.tar.gz https://github.com/opencv/opencv_contrib/archive/4.7.0.tar.gz
tar xf opencv_contrib.tar.gz
rm *.tar.gz
```
We need to temporarily modify two system variables required to successfully build GTK+ support:
```
export PKG_CONFIG_PATH=/usr/lib/arm-linux-gnueabihf/pkgconfig:/usr/share/pkgconfig
export PKG_CONFIG_LIBDIR=/usr/lib/arm-linux-gnueabihf/pkgconfig:/usr/share/pkgconfig
```

## Build
Now we can use CMake to generate the OpenCV build scripts:
(If you use another version of python, don't forget to adjust the python version in the cmake parameter)
```
cd opencv-4.7.0
mkdir build && cd build
cmake -D CMAKE_BUILD_TYPE=RELEASE \
      -D CMAKE_INSTALL_PREFIX=/opt/opencv-4.7.0 \
      -D CMAKE_TOOLCHAIN_FILE=../platforms/linux/arm-gnueabi.toolchain.cmake \
      -D OPENCV_EXTRA_MODULES_PATH=~/opencv_all/opencv_contrib-4.7.0/modules \
      -D OPENCV_ENABLE_NONFREE=ON \
      -D ENABLE_NEON=ON \
      -D ENABLE_VFPV3=ON \
     -D BUILD_TESTS=OFF \
     -D BUILD_DOCS=OFF \
     -D PYTHON2_INCLUDE_PATH=/usr/include/python2.7 \
     -D PYTHON2_LIBRARIES=/usr/lib/arm-linux-gnueabihf/libpython2.7.so \
     -D PYTHON2_NUMPY_INCLUDE_DIRS=/usr/lib/python2/dist-packages/numpy/core/include \
     -D PYTHON3_INCLUDE_PATH=/usr/include/python3.9 \
     -D PYTHON3_LIBRARIES=/usr/lib/arm-linux-gnueabihf/libpython3.9.so \
     -D PYTHON3_NUMPY_INCLUDE_DIRS=/usr/lib/python3/dist-packages/numpy/core/include \
     -D BUILD_OPENCV_PYTHON2=ON \
     -D BUILD_OPENCV_PYTHON3=ON \
     -D BUILD_EXAMPLES=OFF ..
```
Because of the way we installed numpy for python2 before, cmake won't be able to find numpy for python2. To solve this issue, create this link:
```
ln -s  /usr/local/lib/python2.7/dist-packages/numpy/core/include/numpy /usr/include/numpy
```
If anything worked out, you should have a Makefile in the build folder. Now start the actual build. The -j parameter defines the cores used. I noticed, make is prone to errors, the higher the core number. So in case you run into any issues, first try to lower the core number or leave it away.
```
make -j16
```
Once the build phase is done, we can install the library:
```
make install/strip
```
Next, we need to change the name of a library that the installer mistakenly labeled as a x86_64 library when in fact it is an armhf one:
```
cd /opt/opencv-4.7.0/lib/python3.9/dist-packages/cv2/python-3.9/
cp cv2.cpython-39m-x86_64-linux-gnu.so cv2.so
```
Let’s compress the installation folder and save the archive to the home folder:
```
cd /opt
tar -cjvf ~/opencv-4.7.0-armhf.tar.bz2 opencv-4.7.0
cd ~
```
To make our life easier, I’ve also prepared a simple pkg-config settings file, named opencv.pc. Get it with:
```
curl https://raw.githubusercontent.com/XDcobra/CrossCompileOpenCV4PI/main/opencv.pc?token=GHSAT0AAAAAAB5FYDBTBEAAF6H6ICKNUGNGY6IIOUA --output opencv.pc
cp opencv.pc ~
cd ~
```

# Copy Files
Now copy the files from the docker image to the host
```
docker cp root/opencv-4.7.0-armhf.tar.bz2 .
docker cp root/opencv.pc .
```
Now send the files from your host machine to the rasperri pi. I used ssh for example:
```
scp opencv-4.7.0-armhf.tar.bz2 pi@your-pi-ip:/~
scp opencv.pc pi@your-pi-ip:/~
```

# Rasperry Pi setup
Now head over to your rasperry pi. The following commands are all performed on your rasperry pi.
Make sure your RPi has all the development libraries we’ve used. Like before, if you don’t plan to use GUI, ignore the first line from the next commands. Most of these libraries should be already installed if you are using the full version of Raspbian:
```
apt install libgtk-3-dev libcanberra-gtk3-dev
apt install libtiff-dev zlib1g-dev
apt install libjpeg-dev libpng-dev
apt install libavcodec-dev libavformat-dev libswscale-dev libv4l-dev
apt-get install libxvidcore-dev libx264-dev
```
Uncompress and move the library to the /opt folder of your Rasperri pi:
```
tar xfv opencv-4.1.0-armhf.tar.bz2
mv opencv-4.1.0 /opt
rm opencv-4.1.0-armhf.tar.bz2
```
Next, let’s also move opencv.pc where pkg-config can find it:
```
mv opencv.pc /usr/lib/arm-linux-gnueabihf/pkgconfig
```
In order for the OS to find the OpenCV libraries we need to add them to the library path:
```
cd /etc/profile.d/
touch opencvlib.sh
nano opencvlib.sh --> LD_LIBRARY_PATH=/opt/opencv-4.7.0/lib
```
Next, let’s create some symbolic links that will allow Python to load the newly created libraries:
```
ln -s /opt/opencv-4.7.0/lib/python2.7/dist-packages/cv2 /usr/lib/python2.7/dist-packages/cv2
ln -s /opt/opencv-4.7.0/lib/python3.9/site-packages/cv2 /usr/lib/python3/dist-packages/cv2
```
HURRAY! You should be able to use OpenCV 4.7.0 now in Python and C++. In case the libraries are not found, make sure the environment variable is set correctly. You can check with:
```
printenv LD_LIBRARY_PATH
```

## Thank you
The guide is based on an old guide, that won't work anymore. But a lot of steps are used, from it, so thank you:
https://solarianprogrammer.com/2018/12/18/cross-compile-opencv-raspberry-pi-raspbian/
