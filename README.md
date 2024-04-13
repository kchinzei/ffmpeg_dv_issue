# FFmpeg dv Issue

FFmpeg occasionally has an error in certain dv files.
This is my note and intended to test it with sample movie.

- I tested it using ffmpeg version 6.1.1 built with Apple clang version 15.0.0 (clang-1500.3.9.4).
- [`sample_err.dv`](./sample_err.dv) is from my old DV tape, captured and splitted following [Léo Bernard's Blog "Capturing and Archiving MiniDV Tapes on macOS"](https://leolabs.org/blog/capture-minidv-on-macos/).

A simple ffmpeg command to copy input to output

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

Problem is shown as `Assertion cur_size >= size failed at libavutil/fifo.c:2703`.

Misteriously, this does not always happen.
Some dv file goes fine.

### Fail cases

These filter/mapping all fail.  
(Replace `...` in `ffmpeg -i sample_err.dv ... out.dv` with the following).

- `-map 0:v -map 0:a:0 -target ntsc-dv`  
(
The above fail example - copy input to output as NTSC DV.
The sample dv file has two audio streams in 32 kHz sampling.
But NTSC DV target wants one audio stream in 48 kHz sampling.
`0:a:0` maps first audio stream.
Resampling is automatically applied.
)
- `-map 0:v -map 0:a:0 -target ntsc-dv -t 6.08`  
(
Stop copying at 6.08 seconds.
)
- `-i sample_err.mp4 -map 0:v -map 0:a -target ntsc-dv`  
(
Convert mp4 to dv.
You need to change the input to mp4.
)
- `-map 0:v -c:v copy -filter_complex '[0:a:0]aresample=48000[s1]' -map '[s1]'`  
(
Explicitly resample audio to 48kHz using `aresample` filter.
)
- `-f lavfi -i testsrc=r=29.97:s=720x480 -map 1:v -map 0:a:0 -target ntsc-dv -t 10`  
(
Use the generated test pattern as the video stream.
Audio is copied from the input file.
)

### Success cases

These run fine
(Replace `...` of above, again).

- `-map 0:v -map 0:a:0 -target ntsc-dv -t 6.07`  
(
Stop copying at 6.07 seconds.
)
- `-map 0:v -map 0:a:0 -c:v libx264 -c:a ac3 out.mp4`  
(
Convert dv to mp4.
You need to change the output to mp4.
)
- `-map 0:v -c:v copy -filter_complex '[0:a:0]afftdn[s0];[s0]aresample=48000[s1]' -map '[s1]'`  
(
Apply `afftdn` before resampling.
)
- `-f lavfi -i anullsrc=r=48000:cl=stereo -map 0:v -map 1:a -target ntsc-dv -t 10`  
(
Use the generated silent audio as the audio stream.
Video is copied from the input file.
)

## A Workaround

This works

```bash
> ffmpeg -i sample_err.dv -map 0:a:0 -f s16le tmp.wav
> ffmpeg -i sample_err.dv -f s16le -i tmp.wav -map 0:v -map 1:a -target ntsc-dv out.dv
```

The first line writes the audio in tmp.wav.
The second line merges video stream from the dv file and audio in tmp.wav.

## What happens?

Findings are:

- Error is in audio handling.
- It fails [here (line 270 of libavutil/fifo.c)](https://github.com/FFmpeg/FFmpeg/blob/master/libavutil/fifo.c).
- Movies shorter than 6.07 sec escape this error.
The length are ***invariant across erroneous dv files***.

Most likely, some defect in dv file resulted this.
My example is captured from very old - in 1996 - tape,
so it is likely with errors.
I am looking into the source code, but the issue is not found yet.
