# General
https://trac.ffmpeg.org/wiki/Encode/H.264  
https://trac.ffmpeg.org/wiki/Encode/H.265  
https://superuser.com/a/1236387  
https://superuser.com/a/1296511  

GTX 600+ support NVENC H264  
GTX 950+ (GMX20x+) support NVENC HEVC(H.265) (https://en.wikipedia.org/wiki/List_of_Nvidia_graphics_processing_units#GeForce_900_series)  


# Encode with NVENC h264
```bash
ffmpeg -c:v h264_cuvid -i input.mp4 -c:v h264_nvenc -preset llhq -rc:v vbr_minqp -qmin:v 20 -qmax:v 23 -b:v 2500k -maxrate:v 4000k -profile:v high -c:a aac -b:a 128k output.mp4
```
`ffmpeg -h encoder=nvenc_h264`: for description on most of the options.

`-c:v h264_cuvid`: use GPU to decode input file. Specified before `-i` flag, requires knowledge of input file encoder. Can be found out by just running `ffmpeg -i input.mp4`. For x264 it's `h264_cuvid`.

`-c:v h264_nvenc`: use the Nvidia h264 GPU encoder instead of the default CPU. Speeds up the encoding _significantly_. Replace with `nvenc_hevc` for h265 (if supported).

`-preset llhq`: low latency, high quality

`-qmin:v and -qmax:v`: min-max value for quality. 0 is lossless 51 is garbage. Default is 23.

`-b:v 2500k`: target bitrate

# Encode with h265 on CPU
```bash
ffmpeg -i input.mp4 -c:v libx265 -crf 21 -preset slow -c:a aac -b:a 128k output.mp4
```

`-c:v libx256`: use the h265 encoder, much (much) slower on CPU, but a lower size for the same quality. Replace with `libx246` for the h264 encoder.

`-crf 21`: the `crf` flag handles the quality of the output. It guarantees a consistent quality throughout. 0 is lossless, 51 is garbage. Read more at the ffmpeg encoding links at the top.

# Convert to different bitrates and prepare chunks for DASH/HLS streaming
**Important!** Set a constant 2 second GOP size (https://superuser.com/a/1223359
):  
```bash
$GOP_SIZE = $FRAMERATE * 2
```

### **1080p**
```bash
ffmpeg -i input.mp4 -preset slow -codec:a libfdk_aac -b:a 128k -codec:v libx264 -pix_fmt yuv420p -b:v 4500k -minrate 4500k -maxrate 9000k -bufsize 9000k -vf scale=-1:1080 -force_key_frames "expr:eq(mod(n,$GOP_SIZE),0)" -x264opts rc-lookahead=$GOP_SIZE:keyint=2*$GOP_SIZE:min-keyint=$GOP_SIZE output_1080p.mp4
```

### 720p
```bash
 ffmpeg -i input.mp4 -preset slow -codec:a libfdk_aac -b:a 128k -codec:v libx264 -pix_fmt yuv420p -b:v 2500k -minrate 1500k -maxrate 4000k -bufsize 5000k -vf scale=-1:720 -force_key_frames "expr:eq(mod(n,$GOP_SIZE),0)" -x264opts rc-lookahead=$GOP_SIZE:keyint=2*$GOP_SIZE:min-keyint=$GOP_SIZE output_720p.mp4
```

### 480p
```bash
ffmpeg -i input.mp4 -preset slow -codec:a libfdk_aac -b:a 128k -codec:v libx264 -pix_fmt yuv420p -b:v 1000k -minrate 500k -maxrate 2000k -bufsize 2000k -vf scale=854:480 -force_key_frames "expr:eq(mod(n,$GOP_SIZE),0)" -x264opts rc-lookahead=$GOP_SIZE:keyint=2*$GOP_SIZE:min-keyint=$GOP_SIZE output_480p.mp4
```

### 360p
```bash
ffmpeg -i input.mp4 -preset slow -codec:a libfdk_aac -b:a 128k -codec:v libx264 -pix_fmt yuv420p -b:v 750k -minrate 400k -maxrate 1000k -bufsize 1500k -vf scale=-1:360 -force_key_frames "expr:eq(mod(n,$GOP_SIZE),0)" -x264opts rc-lookahead=$GOP_SIZE:keyint=2*$GOP_SIZE:min-keyint=$GOP_SIZE output_360p.mp4
```

Check that the file has correct IDR I-frames (for fps of 25 and GOP size of 2 seconds it should be every 50 frames):
```bash
ffprobe output.mp4 -select_streams v -show_frames -of csv -show_entries frame=coded_picture_number,key_frame,pict_type
```

# Create a preview strip of 1s intervals
Make sure input file has constant GOP SIZE. If GOP_SIZE=2s make sure the intervals are a multiple of 2. Expand `...` to add more/less seconds or increase interval duration. You need to calculate actual values beforehand, based on video duration and required spread:
```bash
ffmpeg -i input.mp4 -an -codec:v libx264 -minrate 300k -maxrate 400k -bufsize 500k -vf scale=352:240,select='between(t\,18\,19)+between(t\,34\,35)+...',setpts=N/FRAME_RATE/TB -t 10 output.mp4
```

# Create a preview strip as above but including audio
Audio frames also need to be specified using the same intervals:
```bash
ffmpeg -i input.MP4 -c:a aac -codec:v libx264 -vf scale=352:240,select='between(t\,18\,19)+between(t\,34\,35)+...',setpts=N/FRAME_RATE/TB -af "aselect='between(t\,18\,19)+between(t\,34\,35)+...',asetpts=N/FRAME_RATE/TB" output.mp4
```

# Sync audio to timestamps
Sync issues may occur on certain platforms when concatenating videos with ffmpeg. Use this during concatenation, or on the output file, to sync to timestamps. Setting async to 1 is the most constricting option and will yield the best results for concat sync issues:
```bash
ffmpeg -i input.mp4 -af aresample=async=1 output.mp4
```

# Overlay a single image over an audio track to create video suitable for upload to Youtube without loss of quality
```bash
ffmpeg -loop 1 -framerate 1 -i <path-to-image> -i <path-to-audio> -c:v libx264 -preset veryslow -crf 0 -c:a copy -shortest output.mkv
```
Optionally add `-vf "scale=1280:720"` to modify viewport size to desired resolution
