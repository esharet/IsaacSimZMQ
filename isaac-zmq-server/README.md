# Example container to mock as Server for Isaac Sim ZMQ Bridge


This example container provides a starting point for building your own server to communicate with Isaac Sim using ZMQ and Protobuf (and optionally MsgPack for client-to-server streaming).
You can use it to run and test your CV models, or any other task that will form a closed loop with Isaac Sim.

The server also provides a GUI to visualize the data sensor messages being recived, using the [DearPyGui](https://github.com/hoffstadt/DearPyGui) library, which is a simple and easy to use and extend.

---

## Instructions

#### Server (Python inside a contatiner)

1. Build the docker image and run it
```bash
cd isaac-zmq-server
./build_server.sh
./run_server.sh
```
2. Inside the container, run the server
```bash
python example.py
```

Optional: Minimal MsgPack-only example (headless, prints stats):
```bash
python example_msgpack.py --port 5561
```

---

## C++ MsgPack server example

The C++ example receives camera frames over ZMQ/MsgPack and can optionally publish control messages (camera control, settings, Franka) back to Isaac Sim as MsgPack.

### Build requirements (on host or inside the container)
- libzmq (runtime and headers)
- msgpack-c++ headers
- OpenCV (core, imgproc, highgui) for the optional viewer

The provided CMake targets will look these up automatically.

### Build
```bash
cd isaac-zmq-server/src/cpp
mkdir build && cd build
cmake ..
make -j$(nproc)  # adjust -j for your CPU
```

### Run
```bash
# Syntax:
# ./msgpack_server <camera_port> <ctrl_cam_port> <settings_port> <franka_port> <publish_control> <enable_gui>
# - Set any control port <= 0 to disable that publisher
# - publish_control: 1 enables all control publishers, 0 disables
# - enable_gui: 1 shows OpenCV window, 0 runs headless

# Typical single-stream run on default camera port with GUI and control publishers enabled
# (from build directory)
./msgpack_server 5561 5557 5559 5560 1 1

# Secondary stream on alternate ports (no control publishers)
./msgpack_server 5591 -1 -1 -1 0 1
```

### Notes for Docker
- If you want the OpenCV window inside the container:
  - On host: `xhost +local:`
  - Run container with display sharing: `-e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix:ro`
  - You may also need GUI backends (GTK). If missing, install: `apt-get install -y libgtk-3-0` inside the container.
- If you prefer headless preview, set the last arg to `0` (disable GUI).

### Matching Isaac Sim ports
- By default, Isaac Sim examples use:
  - Camera stream: 5561
  - Control: 5557 (camera control), 5559 (settings), 5560 (Franka)
- If you change ports in the C++ server, update the mission ports in the Isaac Sim client accordingly.
3. Optional - For the Franka RMPFlow (Multi Camera), start two servers


```bash
# Inside the container
python example.py # server 1 for main camera
# in a second container
python example.py --subscribe_only 1 --port 5591 # server 2 for gripper camera
```
