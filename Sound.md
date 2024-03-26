# Houdini Sound Effects
Houdini works fine for VFX, but the real question is does it work for SFX?

I recently lost my mind and made a bunch of audio effects and a couple of synths in Houdini. Why? No clue!

[Download the HIP file!](hips/sdfs/sdf_volumes.hipnc?raw=true)

<img src="./images/sound/soundfx.png" width="800">

## How it works
Originally it worked with brute force. I made a tool to [convert raw sample data to audio](https://mysterypancake.github.io/Fun/html/rawaudio), which was slow and tedious.

Luckily I found out CHOPS has audio output, so you can mess around and hear the result without leaving Houdini!

The latency is surprisingly good given how many points it has to process, 44100 points for every second of audio!
