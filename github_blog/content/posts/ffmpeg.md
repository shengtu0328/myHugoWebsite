---
title: "FFmpeg"
date: 2022-11-22T13:34:23+08:00 [ffmpeg.md](ffmpeg.md) 
draft: true
---

## 1.mp4转成m3u8及ts文件

#### hls命令

2）并没有严格按照10s 分ts，并且第一秒的视频会不准

/Users/videopls/develop/ffmpeg/ffmpeg -i /Users/videopls/develop/ffmpeg/input.avi -f hls -hls_time 10 -hls_playlist_type vod  -hls_segment_filename /Users/videopls/develop/ffmpeg/output/xrq111%d.ts /Users/videopls/develop/ffmpeg/output/xrq2222.m3u8

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:12
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXTINF:12.500000,
xrq1110.ts
#EXTINF:8.333333,
xrq1111.ts
#EXTINF:5.633333,
xrq1112.ts
#EXT-X-ENDLIST
```



2）严格按照10s 分ts 加了 "expr:gte(t,n_forced*1)"参数

/Users/videopls/develop/ffmpeg/ffmpeg -i /Users/videopls/develop/ffmpeg/input.avi -force_key_frames "expr:gte(t,n_forced*1)" -strict -2 -c:a aac -c:v libx264 -f hls -hls_time 10 -hls_playlist_type vod  -hls_segment_filename /Users/videopls/develop/ffmpeg/output/xrq111%d.ts /Users/videopls/develop/ffmpeg/output/xrq2222.m3u8

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXTINF:10.000000,
xrq1110.ts
#EXTINF:10.000000,
xrq1111.ts
#EXTINF:6.466667,
xrq1112.ts
#EXT-X-ENDLIST


```



#### segment_list命令

这么写也不行，第一秒的视频会不准

FFmpeg 转换命令 : /var/folders/fp/fg__cpln099g08625c1vf0xm0000gn/T/jave/ffmpeg-x86_64-3.2.0-osx -i /Users/videopls/develop/ffmpeg/input.avi -codec:v h264 -codec:a aac -map 0 -f ssegment -segment_format mpegts -segment_list /Users/videopls/develop/ffmpeg/input.avi.m3u8 -segment_time 8 /Users/videopls/develop/ffmpeg/input.avi.%d.ts  





你想知道米的味道吗







## 2.mp4转成m3u8及ts文件（带加密）

/Users/videopls/develop/ffmpeg/ffmpeg 

-i /Users/videopls/develop/ffmpeg/input.avi 

-f hls -hls_time 10 

-hls_key_info_file /Users/videopls/develop/ffmpeg/enctest/enc.keyinfo 

-hls_playlist_type vod 

-hls_segment_filename 

/Users/videopls/develop/ffmpeg/encoutput/xrq111%d.ts 

/Users/videopls/develop/ffmpeg/encoutput/xrq2222.m3u8







用ffplay可以播放

ffplay https://shengtu0328.github.io/posts/ffmpeg/xrq2222.m3u8

https://shengtu0328.github.io/posts/ffmpeg/xrq2222.m3u8

https://shengtu0328.github.io/posts/ffmpeg/xrq1111.ts

https://shengtu0328.github.io/posts/ffmpeg/xrq1112.ts

   




### 参考：https://www.twblogs.net/a/5eec32aa45d52357fb4923e6/?lang=zh-cn

