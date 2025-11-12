# ZMQ MsgPack Quickstart

This note covers three common ways to stream camera frames from Isaac Sim to a
MsgPack ZMQ subscriber using the new `ZMQMsgpackAnnotator`:

1. Launch Isaac Sim normally and paste a small helper into the Script Editor.
2. Start Isaac Sim from the command line with a utility script.
3. Drive the same flow in headless mode.

The examples below assume you already built the project (`./build.sh`) and that
`simple_msgpack_camera_gui.py` (or any other subscriber) is running to verify
frames.

---

## 1. Use the Script Editor inside Isaac Sim

1. Open the stage that contains your camera (e.g. `/home/user/omniverse/is40/zmq-turtle-rate-camera.usd`).
2. Open **Window → Script Editor**.
3. Paste the snippet below, adjust the camera path, resolution, topic, etc., and
   press **Run**.

```python
import asyncio
import omni.kit.app
import omni.timeline

from isaacsim.zmq.bridge.examples.core.ZMQMsgpackAnnotator import ZMQMsgpackAnnotator

CAMERA_PATH = "/World/turtlebot3_burger/base_link/car_camera"
RESOLUTION = (1280, 720)
ANNOTATOR_TOPIC = "camera/image"
ZMQ_IP = "0.0.0.0"          # bind on all interfaces
ZMQ_PORT = 5561

annotator = ZMQMsgpackAnnotator(
    camera_path=CAMERA_PATH,
    resolution=RESOLUTION,
    ip=ZMQ_IP,
    port=ZMQ_PORT,
    topic=ANNOTATOR_TOPIC,
)

app = omni.kit.app.get_app()
timeline = omni.timeline.get_timeline_interface()
subscription = None

async def publish_frame(_dt):
    await annotator.publish()

def on_update(event):
    if timeline.is_playing():
        asyncio.ensure_future(publish_frame(event.payload["dt"]))

def start():
    global subscription
    if subscription is None:
        subscription = app.get_update_event_stream().create_subscription_to_pop(on_update)
        print("MsgPack annotator started")

def stop():
    global subscription
    if subscription is not None:
        subscription = None
        annotator.destroy()
        print("MsgPack annotator stopped")

start()
```

4. Press **Play** on the timeline. Frames are now emitted on `tcp://0.0.0.0:5561`
   with topic `camera/image`.
5. Run the viewer (from the repo root) to confirm:

```bash
python isaac-zmq-server/src/simple_msgpack_camera_gui.py \
    --ip 127.0.0.1 --port 5561 --topic camera/image \
    --width 1280 --height 720
```

(Press `ESC` to close the window.)

When you are done, call `stop()` in the Script Editor or simply restart Isaac
Sim.

---

## 2. Launch Isaac Sim with a helper script

A convenience launcher is provided at `tools/run_zmq_msgpack.py`. It wraps the
usual Isaac Sim startup flags, loads a USD stage, and executes a helper Python
script from the repo.

```
python tools/run_zmq_msgpack.py \
    --usd /home/user/omniverse/is40/zmq-turtle-rate-camera.usd
```

By default, it uses `exts/isaacsim.zmq.bridge.examples/isaacsim/zmq/bridge/examples/scripts/zmqpublish.py`.
You can override with `--script` to use a different script (e.g., `zmqpublish_direct.py` for when the USD is already loaded).

Useful options:

- `--launcher` — override the default `/home/user/Downloads/latest_isaacsim_4.5/isaac-sim.sh`.
- `--headless` — automatically switch to `isaac-sim-headless.sh` if it exists.
- `--extra ...` — forward additional arguments to the launcher (e.g. GPU options).

Once the window appears, the USD stage is loaded and the script executes,
instantiating the MsgPack annotator automatically. Use the same viewer command as
above to monitor frames.

---

## 3. Headless mode

The same launcher can start Isaac Sim without a GUI. Add `--headless` to switch
from `isaac-sim.sh` to `isaac-sim-headless.sh` (the script checks the sibling
binary automatically):

```
python tools/run_zmq_msgpack.py \
    --headless \
    --usd /home/user/omniverse/is40/zmq-turtle-rate-camera.usd
```

In headless mode the MsgPack annotator runs exactly as in the GUI case, so you
can keep the same ZMQ subscriber. If you need to tweak simulation parameters or
log information, edit the script in `exts/isaacsim.zmq.bridge.examples/isaacsim/zmq/bridge/examples/scripts/`.

---

## Subscribers and Debugging

Two simple clients are included under `isaac-zmq-server/src/`:

- `simple_msgpack_camera_gui.py` — DearPyGui viewer (no OpenCV dependencies).
- `msgpack_camera_sub.py` — OpenCV-based script (requires an X/Qt environment).

Example GUI viewer usage:

```
python isaac-zmq-server/src/simple_msgpack_camera_gui.py \
    --ip 127.0.0.1 --port 5561 --topic camera/image \
    --width 1280 --height 720
```

If you prefer the OpenCV script, run it on a machine with a GUI stack (or use a
headless OpenCV build and save frames to disk).

---

## Summary

- Use the Script Editor for quick experiments (Section 1).
- Use `tools/run_zmq_msgpack.py` to launch Isaac Sim with the MsgPack publisher
  and your USD stage (Section 2).
- Add `--headless` to run without a GUI (Section 3).
- Subscribe with either of the provided scripts to verify the stream.

The `ZMQMsgpackAnnotator` class is reusable—import it in your own missions or
extensions whenever you need a Gazebo-style MsgPack feed from Isaac Sim.
