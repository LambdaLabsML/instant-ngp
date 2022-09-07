# Guide

## Installation

### Step 0: Install CMake (tested with 3.24.1)


`apt` installed CMake is too old for this project. We build the latest version from source by following the guide [here](https://cmake.org/install/):

Download and unzip the [source](https://github.com/Kitware/CMake/releases/download/v3.24.1/cmake-3.24.1.tar.gz).

```
$ sudo apt-get install libssl-dev
$ cd cmake-3.24.1
$ ./bootstrap
$ make
$ make install
```

Verify you have `cmake > 3.21`:

```
$ which cmake
/usr/local/bin/cmake
$ cmake --version
cmake version 3.24.1
```

### Step 1: Install some extra packages

```
$ sudo apt-get install \
    build-essential \
    python3-dev \
    python3-pip \
    libopenexr-dev \
    libxi-dev \
    libglfw3-dev \
    libglew-dev \
    libomp-dev \
    libxinerama-dev \
    libxcursor-dev
```

### Step 2: Install OptiX to /usr/local

Choose “Linux Accept & Download” from [here](https://developer.nvidia.com/designworks/optix/download).

```
$ chmod +x NVIDIA-OptiX-SDK-7.5.0-linux64-x86_64.sh
$ ./NVIDIA-OptiX-SDK-7.5.0-linux64-x86_64.sh
$ sudo mv NVIDIA-OptiX-SDK-7.5.0-linux64-x86_64 /usr/local
```

Note: it seems enough to just move `NVIDIA-OptiX-SDK-7.5.0-linux64-x86_64` to `/usr/local` so that the `include` [folder](https://github.com/NVlabs/instant-ngp/blob/7fa1b93d326e1dafa650d99bb9ed862f24c3dfd5/cmake/FindOptiX.cmake#L62) can be found while building instant ngp. You do not need to actually build the OptiX library itself. 

### Step 3: Install Vulkan SDK for DLSS support

DLSS option only available if Vulkan SDK is installed. The installation guide is from [here](https://vulkan.lunarg.com/sdk/home#linux):

```
wget -qO - https://packages.lunarg.com/lunarg-signing-key-pub.asc | sudo apt-key add -
sudo wget -qO /etc/apt/sources.list.d/lunarg-vulkan-1.3.224-focal.list https://packages.lunarg.com/vulkan/1.3.224/lunarg-vulkan-1.3.224-focal.list
sudo apt update
sudo apt install vulkan-sdk
```

### Step 4: Add CUDA to PATH

Add the following lines to `~/.bashrc`, then run `source ~/.bashrc`.

```
export PATH="/usr/bin:$PATH" 
export LD_LIBRARY_PATH="/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH"
```


### Step 5: Install Colmap
Largely follow the instructions [here](https://colmap.github.io/install.html#linux), except that __we SKIPPED `CMake`__, (which has already been installed earlier with a newer version):

```
$ sudo apt-get install \
    libboost-program-options-dev \
    libboost-filesystem-dev \
    libboost-graph-dev \
    libboost-system-dev \
    libboost-test-dev \
    libeigen3-dev \
    libsuitesparse-dev \
    libfreeimage-dev \
    libmetis-dev \
    libgoogle-glog-dev \
    libgflags-dev \
    libglew-dev \
    qtbase5-dev \
    libqt5opengl5-dev \
    libcgal-dev
```
Install Ceres Solver:

```
$ sudo apt-get install libatlas-base-dev libsuitesparse-dev
$ git clone https://ceres-solver.googlesource.com/ceres-solver
$ cd ceres-solver
$ git checkout $(git describe --tags) # Checkout the latest release
$ mkdir build
$ cd build
$ cmake .. -DBUILD_TESTING=OFF -DBUILD_EXAMPLES=OFF
$ make -j
$ sudo make install
```

Configure and compile COLMAP:

```
$ git clone https://github.com/colmap/colmap.git
$ cd colmap
$ git checkout dev
$ mkdir build
$ cd build
$ cmake ..
$ make -j
$ sudo make install
```

### Step 6: Install instant NeRF

Clone the repo __recursively__:

```
$ git clone --recursive https://github.com/LambdaLabsML/instant-ngp
$ cd instant-ngp
$ cmake . -B build
$ cmake --build build --config RelWithDebInfo -j
```
Note the last step may be stuck at around 52% for a while (can be ~10 mins). Just be patient.


## Test

### Step 7: Simple Test

Run the fox demo from NVIDIA.
```
$ ./build/testbed --scene data/nerf/fox
```

### Step 8: Create your own data

You can download an example video (shot with an iphone) and put it in the `data/nerf/lambda/lego` folder. The following command can extract frames from the video, and calibrates camera positions with colmap.
```
$ wget https://lambdalabs-files.s3.us-west-2.amazonaws.com/mlteam/nerf/data/lego/IMG_0592.MOV
$ mv IMG_0592.MOV data/nerf/lambda/lego
$ python scripts/colmap2nerf.py \
  --video_in data/nerf/lambda/lego/IMG_0592.MOV \
  --out data/nerf/lambda/lego/transforms.json
  --video_fps 2 --run_colmap --aabb_scale 16
```

Run Nerf on the folder of images
```
$ ./build/testbed --scene data/nerf/lambda/lego
```

### Step 9: Render video

In order to render video one needs additional Python packages in the `requirements.txt` file. It is generally a good practice to create a virtualenv for installing those packages:

```
virtualenv -p /usr/bin/python3.8 .venv
. .venv/bin/activate
pip install -r requirements.txt
```

The general practice of creating a flythrough video is to 
- Run testbed for a while and save the NeRF snapshot (the latest model status) as `base.msgpack`
- Set camera key frames in the "Camera path" GUI, and save the camera path as `base_cam.json`. 
- Run `scripts/run.py` script which loads the snapshot and camera path, and render the scene.

This is an example command that generates a flythrough video from the lego scene.

```
CUDA_VISIBLE_DEVICES=1 python scripts/run.py --mode nerf \
--scene data/nerf/lambda/lego/ --load_snapshot data/nerf/lambda/lego/base.msgpack \
--video_camera_path data/nerf/lambda/lego/base_cam.json --video_n_seconds 5 \
--video_fps 30 --width 1024 --height 576 \
--video_output data/nerf/lambda/lego/output.mp4 \
--min_x -0.028 \
--min_y 0.115 \
--min_z -0.160 \
--max_x 1.028 \
--max_y 0.885 \
--max_z 0.946
```

We added options for AABB cut (`min_x`, `min_y`, `min_z`, `max_x`, `max_y`, `max_z`). However DLSS is [hard to enable](https://github.com/NVlabs/instant-ngp/discussions/824) via the offline rendering.

This is the result:

<img src="data/nerf/lambda/lego/lego_output.gif" height="342"/>