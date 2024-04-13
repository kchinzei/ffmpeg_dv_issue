# FFmpeg dv Issue

FFmpeg has error in certain .dv file.
This is my note and intended to test it with sample movie.

[`sample_err.dv`](./sample_err.dv) is from my old DV tape, captured and splitted according to [Léo Bernard's Blog "Capturing and Archiving MiniDV Tapes on macOS"](https://leolabs.org/blog/capture-minidv-on-macos/).

Simplest `ffmpeg` command

```bash
> ffmpeg -i sample_err.dv -map 0:v -map 0:a:0 -target ntsc-dv out.dv
```
will end up with

```bash
[dv @ 0x14e606880] Estimating duration from bitrate, this may be inaccurate
...
Assertion cur_size >= size failed at libavutil/fifo.c:2703 bitrate=   0.0kbits/s speed= 154x    
Abort trap: 6
```

This does not always happen.
Some dv file goes fine.

### Fail cases

These filter/mapping all fail.  
(Replace `...` in `ffmpeg -i sample_err.dv ... out.dv` with the following).

- `-map 0:v -map 0:a:0 -target ntsc-dv -t 6.08`
- `-map 0:v -c:v copy -filter_complex '[0:a:0]aresample=48000[s1]' -map '[s1]'`
- `-f lavfi -i testsrc=r=29.97:s=720x480 -map 1:v -map 0:a:0 -target ntsc-dv -t 10`

### Success cases

These run fine
(Replace `...` of above, again).

- `-map 0:v -map 0:a:0 -target ntsc-dv -t 6.07`
- `-map 0:v -c:v copy -filter_complex '[0:a:0]afftdn[s0];[s0]aresample=48000[s1]' -map '[s1]'`
- `-f lavfi -i anullsrc=r=48000:cl=stereo -map 0:v -map 1:a -target ntsc-dv -t 10`

## A Workaround

This works

```bash
> ffmpeg -i sample_err.dv -map 0:a:0 -f s16le tmp.wav
> ffmpeg -i sample_err.dv -f s16le -i tmp.wav -map 0:v -map 1:a -target ntsc-dv out.dv
> rm tmp.wav
```

The first line write the audio in tmp.wav.
The second line merges video channel from the dv file and audio channel from tmp.wav.

## What happens?

Findings are:

- Error is in audio handling.
- Movies shorter than 6.08 sec escape this error.
The length are **invariant across erroneous dv files**.

I am looking into the source code, but the issue is not found yet.
