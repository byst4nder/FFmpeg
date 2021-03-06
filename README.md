# FFmpeg doc

## FFmpeg

### A.V.S.D.C

示例：

-crf 18 -preset 8 -tune film -profile:v high -level 51 -pix_fmt yuv420p -maxrate 24M -bufsize 48M -refs 4 -bf 6 -r 30000/1001 -s 1440x1080 -g 290 -keyint_min 1 -fast-pskip 0 -me_method umh -me_range 32 -subq 10 -aq-mode 2 -aq-strength 0.9 -trellis 2 -psy-rd 0.8:0.05 -ar 48000 -b:a 256k -ac 2 -c:a aac -af loudnorm -max_muxing_queue_size 2222




- **批处理**

```
for %a in (*.mp4 *.flv) do ffmpeg -i "%a" -crf 20 "output\%~na_cfr-20.mp4"  
Get-ChildItem *.jpg | ForEach-Object { ffmpeg -hide_banner -i $_.Name -lossless 1 "$($_.BaseName).webp" }
```

> 先建立output文件夹；若要保存为.bat，则需将%替换为%% 




- **切片与拼接**

```
ffmpeg -y -hide_banner -i .\simple.mov -c copy -f segment -segment_time 13 -reset_timestamps 1 -segment_list simple.ffcat simple_%3d.mov  
ffmpeg -y -hide_banner -i .\simple.ffcat -c copy .\simple-c.mov  
p.py -p -s audio .\simple-c.mov
```

> 适合 MPEG CFR  
> http://ffmpeg.org/ffmpeg-formats.html#segment_002c-stream_005fsegment_002c-ssegment




- **合并多条视频分段**

```
(for %i in (*.flv) do @echo file '%i') > mylist.txt  
Get-ChildItem *.mp4 | ForEach-Object { Write-Output "file '$($_.Name)'" } | Out-File mylist.txt  
```

```
ffmpeg -f concat -i mylist.txt -c copy YYY.mkv
```

> https://trac.ffmpeg.org/wiki/Concatenate




- **提取视频片段**

```
ffmpeg -ss 00:00:18.000 -i XXX.mp4 -t 10 -c copy -avoid_negative_ts 1 YYY.mp4
```

> 快速寻址；从上一关键帧开始裁切 17.345s~28.000s

> 若`XXX.mp4 start: 0.000000`，则 `ss to i` 与 `ss i t` 等效 

> https://trac.ffmpeg.org/wiki/Seeking     

|                          |               优点                |                     缺点                      |
| :----------------------: | :-------------------------------: | :-------------------------------------------: |
| 有 `-c copy`，不重新编码 |             瞬间提取              |             实际时间范围不精确             |
| 无 `-c copy`，需重新编码 | 实际时间范围准确：18.000s~28.000s |             重编码缓慢，CPU满负载             |




- **分离视频流与音频流**

```
ffmpeg -i input.mkv -map 0:1 -map 0:2 -c copy audios_only.mkv -map 0:0 -c copy video_only.mkv
```




- **循环流**

```
ffmpeg -stream_loop 3 -i XXX.wav -c copy XXX_x4.wav
```

> `-stream_loop number (input)`  
> Set number of times input stream shall be looped. Loop 0 means no loop, loop -1 means infinite loop.




- **合并视频与音频**

```
ffmpeg -i XXX.mp4 -i XXX.aac -c copy YYY.mp4
```

> 合并后时长取较长段。若视频较长，则后半段音量为零；若音频较长，则后半段为视频的最后一帧。




- **旋转视频方向**

```
ffmpeg -i XXX.mp4 -map_metadata 0 -metadata:s:v rotate="90" -c copy YYY.mp4
```




- **裁切视频**

```
ffmpeg -i XXX.mp4 -vf crop=w:h:x:y,scale=3840:-2 YYY.mp4
```




- **加速视频和音频**

```
ffmpeg -i 30fps.mp4 -lavfi "setpts=0.5*PTS;atempo=2" -r 60 60fps.mp4
```

> https://trac.ffmpeg.org/wiki/How%20to%20speed%20up%20/%20slow%20down%20a%20video




- **录制桌面**

CPU:

```
ffmpeg -y -probesize 64M -f gdigrab -framerate 30 -i desktop -qp 0 -preset 0 -level 51 YYY.mp4
```

```
ffmpeg -thread_queue_size 512 -f gdigrab -framerate 30 -i desktop -f dshow -i audio="立体声混音 (Realtek(R) Audio)" -qp 0 -preset 0 -level 51 YYY.mp4
```

GPU:

```
ffmpeg -f gdigrab -framerate 30 -i desktop -c:v h264_nvenc -qp 0 -profile:v high -level 51 YYY.mkv
```

> https://trac.ffmpeg.org/wiki/Capture/Desktop  
> https://ffmpeg.org/ffmpeg-devices.html#gdigrab  
> GPU录制：总功耗46w；CPU录制：总功耗25w；皆无法正常录制60帧

同时录制声音:

```
ffmpeg -list_devices true -f dshow -i dummy     
ffmpeg -f gdigrab -framerate 30 -i desktop -f dshow -i audio="麦克风阵列 (Realtek(R) Audio)" YYY.mp4  
ffmpeg -f gdigrab -framerate 30 -i desktop -f dshow -i audio="立体声混音 (Realtek(R) Audio)" YYY.mp4
```   









### Video

- **Nvidia GPU NVENC 编码**

```
ffmpeg -y -stats -v 24 -ss 5 -i XXX.mp4 -t 8 -an -level 52 -r 60 -g 300 -keyint_min 1 -b:v 0 -bufsize 0 -c:v h264_nvenc -preset slow -profile:v high -rc-lookahead 32 -bf 4 -b_ref_mode 2 -temporal_aq 1 -spatial_aq 1 -aq-strength 15 -qp 16 YYY.mp4
```

'...... -hwaccel cuda -c:v h264_cuvid -i XXX.mp4 ......': 解码加快10%左右，但输出帧数量与源不一致

> https://github.com/Xaymar/obs-StreamFX/wiki/Encoder-FFmpeg-NVENC

> https://devblogs.nvidia.com/nvidia-ffmpeg-transcoding-guide/




- **GIF 转 Animated WebP**

```
ffmpeg -i XXX.gif -vf scale=320:-1,fps=15 -loop 0 -lossles 1 -y YYY.webp
```




- **mp4 转 gif**

```
ffmpeg -ss 5 -i XXX.mp4 -t 5 -vf fps=10,scale=480:-2 -loop 0 YYY.gif
```

高质量版:    

```
ffmpeg -i XXX.mp4 -vf "fps=10,scale=480:-1:flags=lanczos,split[s0][s1];[s0]palettegen[p];[s1][p]paletteuse" -loop 0 YYY.gif
```




- **添加文字水印**

```
ffmpeg -i XXX.mp4 -vf "drawtext=font=consolas:text='Abg0123':x=20:y=H-th-20:fontsize=30:fontcolor=white:borderw=3:bordercolor=black" YYY.mp4
```




- **生成单色视频并添加计时器水印**

```
ffmpeg -hide_banner -y -f lavfi -i "color=c=0x333333:s=1920x1080:r=10,drawtext=fontfile=C\\:/Windows/fonts/consola.ttf:fontsize=96:fontcolor='white':timecode='00\:00\:00\:00':rate=10:text='TCR\:':boxcolor=0x000000AA:box=1:x=960-text_w/2:y=540" -preset 3 -g 100 -keyint_min 100 -t 60 YYY.mp4
```

> rate = video fps

[效果图](https://i.loli.net/2019/10/02/B8NfrWOpSjwFVc2.png)




- **周期性显示文字水印（周期1.6s，显示0.8s）**

```
ffmpeg -i XXX.mp4 -vf "drawtext=font=consolas:text='test0test':x=100:y=100:enable=lt(mod(t\,1.6)\,0.8):fontsize=30:fontcolor=blue" -y YYY.mp4
```




- **添加图片水印**

```
ffmpeg -i XXX.mp4 -i XXX.png -filter_complex overlay=20:20:enable='between(t,10,16)' -preset 0 YYY.mp4
```

右上：overlay=W-w-20:20  
右下：overlay=W-w-20:H-h-20  
左下：overlay=20:H-h-20  
居中：overlay=(W-w)/2:(H-h)/2  




- **添加覆盖动画（从t=5s开始，速度400，位置正中）**

```
ffmpeg -i XXX.mp4 -i XXX.png -filter_complex "overlay='if(gte(t,5), -w+(t-5)*400, NAN)':(H-h)/2" YYY.mp4
```




- **添加Gif图片水印**

```
ffmpeg -i XXX.mp4 -ignore_loop 0 -i XXX.gif -filter_complex overlay=20:20:shortest=1 YYY.mp4
```




- **跑马灯**

```
ffmpeg -i XXX.mp4 -i XXXX.mp4 -filter_complex "overlay=x='if(gte(t,2), -w+(t-2)*400, NAN)':y=0" -s 1920x1080 YYY.mp4
```





- **导出图片并合并**

```
ffmpeg -ss 10 -i XXX.mp4 -frames:v 1 YYY.png
```

```
ffmpeg -ss 10 -i XXX.mp4 -frames:v 120 YYY_%3d.png
```

```
ffmpeg -framerate 30 -i YYY_%3d.png -c copy YYY.mkv
```




- **创建仅包含图像的视频**

```
ffmpeg -loop 1 -framerate FPS -t 5 -i XXX.png -c:v libx264 -pix_fmt yuv420p -qp 0 -preset 0 YYY.mp4
```




- **HEVC.4K.HDR.10bit >>> x264.1080p.SDR.8bit**

```
-vf zscale=t=linear:npl=100,format=gbrpf32le,zscale=p=bt709,tonemap=tonemap=hable:desat=0,zscale=t=bt709:m=bt709:r=tv,format=yuv420p,zscale=1920:-2
```




- **编码为Prores**

`ffmpeg -h encoder=prores_ks`

```
ffmpeg -i XXX.mp4 -c:v prores_ks -profile:v 4 -pix_fmt yuva444p10le -c:a pcm_s16le YYY.mov
```

> https://video.stackexchange.com/questions/14712/how-to-encode-apple-prores-on-windows-or-linux  
> https://trac.ffmpeg.org/wiki/Encode/VFX  
> https://wideopenbokeh.com/AthenasFall/?p=111  
> 相比于prores_ks, prores牺牲了压缩率提升了编码速度（profile:v也适用于prores）









### Audio

- **查询音量**

【RMS】:

```
ffmpeg -i XXX.mp4 -af volumedetect -f null nul
```

【EBU R128】:

```
ffmpeg-normalize xxx.wav -p -n
```

```
ffmpeg -hide_banner -i XXX.mp4 -map a:0 -af ebur128=peak=true:framelog=verbose -f null -
```

> https://ffmpeg.org/ffmpeg-filters.html#ebur128-1




- **正常化音量大小**

```
ffmpeg-normalize audio.m4a -vn -sn -mn -cn -t -17 -tp -1 -lrt 15 -o output.wav
```

> https://github.com/slhck/ffmpeg-normalize




- **调节音量（10~16s音量为150%）**

```
ffmpeg -i XXX.mp4 -c:v copy -af volume=1.5:enable='between(t,10,16)' YYY.mp4
```

> Volume与CRF算法相似，volume=0.5 相当于 volume=-6dB




- **绘制音频波形图**


```
ffmpeg -i XXX.wav -filter_complex "showwavespic=s=1920x1080:split_channels=1" -frames:v 1 YYY.png
```

[【示例图】](https://github.com/antatura/FFmpeg/blob/master/Images/output.png)

> https://trac.ffmpeg.org/wiki/Waveform




- **绘制音频频谱**

```
ffmpeg -i XXX.aac -lavfi showspectrumpic=s=1764x1024:mode=separate:color=terrain spectrogram.png
```

[【示例图】](https://github.com/antatura/FFmpeg/blob/master/Images/spectrogram-q.png)

https://ffmpeg.org/ffmpeg-filters.html#showspectrumpic

> 高度需为2的幂次方




- **本地aac高质量编码**

```
ffmpeg -i XXX.aac -ar 48000 -b:a 256k -aac_coder 1 -strict -2 -cutoff 24000 YYY.aac
```









### Subtitle

- **导出字幕**

```
ffmpeg -i XXX.m2ts -map 0:2 -c:s copy YYY.sup
```

- **烧制字幕**

```
ffmpeg -i XXX.mp4 -vf subtitles=XXX.srt YYY.MP4
```

`ass=XXX.ass`









## FFprobe

- **查看视频Info**

```
ffprobe -v error -show_format -show_streams XXX.mp4
```




- **输出每一帧的 time, size, type**

```
ffprobe -v error -select_streams v:0 -show_entries frame=pkt_pts_time,pkt_size,pict_type -of csv=p=0 XXX.mp4 >XXX.csv
```




- **获取所有关键帧** 

```
ffprobe -loglevel error -select_streams v:0 -skip_frame nokey -show_entries frame=pkt_pts_time -of csv=print_section=0 XXX.mp4
```




- **关键帧计数**

```
ffprobe -v error -count_frames -select_streams v:0 -skip_frame nokey -show_entries stream=nb_read_frames -of default=nokey=1:noprint_wrappers=1 XXX.mp4
```

`-skip_frame nokey`: Keyframes

`-skip_frame nointra`: I frames

`-skip_frame bidir`: except B frames

> https://stackoverflow.com/questions/2017843/fetch-frame-count-with-ffmpeg




- **获取01:25:31前一个关键帧**

```
ffprobe -i XXX.mkv -show_frames -read_intervals 01:25:31%+#1
```




- **为MP3导入元数据和封面**

```
ffmpeg -i XXX.mp3 -i XXX.png -map 0:0 -map 1:0 -c copy -id3v2_version 3 -write_id3v1 1 -metadata title="?" -metadata artist="?" -metadata album="?" -metadata comment="Cover (front)" YYY.mp3
```









## FFplay 

## **视频差值对比**

```
ffplay -f lavfi "movie=XXX.mp4[a];movie=YYY.mp4[b];[a][b]blend=all_mode=difference,eq=gamma=1.5,hue=h=312"
```

> https://ffmpeg.org/ffmpeg-filters.html#blend-1









## metaflac

- **编辑FLAC元数据与封面**

```
metaflac --remove-all XXX.flac

metaflac --import-tags-from=FlacTags.txt --import-picture-from=cover.jpg XXX.flac
```

[FlacTags.txt](https://github.com/antatura/FFmpeg/blob/master/Examples/FlacTags.txt) (ANSI编码)

> https://xiph.org/flac/documentation_tools_metaflac.html

[Vorbis注释规范](https://xiph.org/vorbis/doc/v-comment.html)









## Powershell x FFmpeg

- **Powershell生成缩略图**

```
$file=Read-Host "Input File"; $d=[double](ffprobe -v 16 -show_entries format=duration -of csv=p=0 $file); for ($E=1; $E -le 5; $E++) {$ss=$E*$d/6; ffmpeg -y -v 16 -ss $ss -i $file -frames:v 1 "tmp_$E.png"}; ffmpeg -y -v 16 -i tmp_%d.png -vf scale=1920:-2,tile=1x5:padding=32 otpt.png;
```











