# Raspberry Pi Surveillance Server and Client Machines

This is meant to showcase how one can copy over the python file in this repo to a Raspberry Pi 5 device with an attached camera and
be able to capture the video that is being output on another device as an elementary surveillance system.

This is also meant for my own personal use.

Consider these notes and files as a starting off point.

All assumptions below are that you are able to SSH into the RPI Server.

## Setting up the RPI Server

### Copy the `rpi_server_camera.py` onto the RPI server

```bash
scp rpi_server_camera.py <user>@<ip-address>:/home/<user>/rpi_server_camera.py
```

### Systemd Configuration

Modify the `rpi_server.service` file so that `<youruser>`,`<yourgroup>` is replaced with your own user/group name on the server.

```bash
scp rpi_server.service <user>@<ip-address>:/home/<user>/etc/systemd/system/rpi_server_camera.service
ssh <user>@<ip-address>
sudo systemctl daemon-reload
sudo systemctl enable rpi_server
sudo systemctl start rpi_server
```

Now even on reboot, the python program should be running as expected.

## Setting up the Client capture

### The parent `bash` command

For a time, we will want to only capture footage from the camera for a certain amount of time to prevent saving too much data onto a harddrive and running out of space.

This command will be run on the Client Server to capture the footage from the RPI Server

```bash
# Define duration for timeout (3 hours in seconds)
$ DURATION=$((3 * 60 * 60))
$ timeout $DURATION # <ffmpeg-commands> - see below for the cli command string to implement
```

### `ffmpeg` configuration options

- `-i <stream-url>` captures the incoming stream
- `an` removes the audio
- `c:v libx264` video codec definition
- `crf 23` Constant Rate Factor (CRF) controls the quality. Lower is better, but uses more space. Common range is 18-to-28
- `-preset slow` processes the video to compress it to save space
- `-reconnect_streamed 1`, `-reconnect_delay_max 5` enable reconnection features
- `-f segment` split the output into multiple segments
- `-segment_time 5` splits the segments after a duration of time (number means seconds, but I have noticed that the actual length of the clip is x2. 5 = 10 seconds in actual video time).
  (
  The nature of the segment video splits not being exact is well known: https://ffmpeg.org/ffmpeg-formats.html > "Note that splitting may not be accurate, unless you force the reference stream key-frames at the given time. See the introductory notice and the examples below."
  )

- `-reset_timestamps 1` resets timestamps at the begnning of each segment to start from 0
- `-strftime 1 "%Y-%m-%d_%H-%M-%S_output.mp4"` converts the output file's filename to something like "2024-01-07_15-48-18_output.mp4"

### Examples

#### H.264 - NVENC codec

If the system has an onboard NVIDIA graphics card, we can use the NVIDIA codec to do the footage capturing instead in order to save CPU cycles.

```
ffmpeg -i http://<ip-address>:8000/stream.mjpg \                                              148 ↵
-an \
-c:v h264_nvenc \
-crf 23 \
-preset slow \
-reconnect_streamed 1 \
-reconnect_delay_max 5 \
-f segment \
-segment_time 5 \
-reset_timestamps 1 \
-strftime 1 "%Y-%m-%d_%H-%M-%S_output.mp4"
```

Current output per segment based on the above configuration is ~2MB for 10 seconds (`-segment_time 5`)

#### H.265

```
ffmpeg -i http://<ip-address>:8000/stream.mjpg \
-an \
-c:v libx265 \
-crf 23 \
-preset slow \
-reconnect_streamed 1 \
-reconnect_delay_max 5 \
-f segment \
-segment_time 5 \
-reset_timestamps 1 \
-strftime 1 "%Y-%m-%d_%H-%M-%S_output.mp4"
```

#### Output times based on encoding

Current output per segment based on the above configuration is ~1.5MB for 10 seconds (`-segment_time 5`)

| Durations | H264  | H265  | CRF |
| :-------: | :---: | :---: | :-: |
|  1 Hour   | 120MB | 360MB | 23  |
|  3 Hour   | 90MB  | 270MB | 23  |
