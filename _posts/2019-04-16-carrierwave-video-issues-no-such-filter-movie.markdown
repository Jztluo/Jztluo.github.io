---
layout: post
title:  "carrierwave-aliyun 报错:No such filter movie"
date:   2019-04-16 14:34:52 +0800
categories: rails carrierwave
---
# 问题
- rails 5.2.2.1
- carrierwave 1.3.1
- carrierwave-video 0.6.0
- streamio-ffmpeg 3.0.2

在使用 carrierwave-video 为视频添加水印的时候出现了以下异常
```
[AVFilterGraph @ 0x7fe8005aaf80] No such filter: '"movie'
Error reinitializing filters!
Failed to inject frame into filter network: Invalid argument
Error while processing the decoded data for stream #0:0
Conversion failed!

Errors: encoded file is invalid.
```

日志往上翻，调用的命令为
```
E, [2019-04-04T14:14:56.776727 #23269] ERROR -- : Failed encoding...
["/usr/local/bin/ffmpeg", "-y", "-i", "******/tmp/1554358372-23269-0001-4130/page-transformer.mp4", "-vcodec", "libx264", "-acodec", "aac", "-s", "640x1138", "-r", "30", "-strict", "-2", "-map_metadata", "-1", "-vf", "\"movie=******/watermark.png [logo]; [in][logo] overlay= [out]\"", "-aspect", "0.562390158172232", "******//tmp/1554358372-23269-0001-4130/tmpfile.mp4"]
```

拼成字符串后，用命令行执行又可以成功。

#  原因

carrierwave-video issues 没有人和我一样的问题

有人说是版本问题:
https://stackoverflow.com/questions/10455611/how-to-solve-ffmpeg-watermark-no-such-filter-movie-and-failed-to-avformat

这有个类似的问题: 
https://stackoverflow.com/questions/20504125/ffmpeg-use-filter-in-android

猜测可能是由于部分(少数?)系统环境的原因, 导致生成的命令中会多出引号: `'"movie'`.

生成有关参数的源码在`CarrierWave::Video::FfmpegOptions`中:
```ruby
def watermark_params
  return [] unless watermark?

  @watermark_params ||= begin
          
    ...

    ["-vf", "\"movie=#{path} [logo]; [in][logo] overlay=#{positioning} [out]\""]
  end
end
```
看来一开始就会加上引号，在之后在几次调用中 ` FFMPEG::Movie#transcode => FFMPEG::Transcoder#transcode_movie => Open3.popen3` 参数也没有被改变.

# 解决

在一开始的时候去掉引号，就能跑通了. 至于会不会导致其他环境出问题就不深究了.
```ruby
module FixWatermarkParams
  def watermark_params
    return [] unless watermark?

    @watermark_params ||= begin
                            path = @format_options[:watermark][:path]
                            position = @format_options[:watermark][:position].to_s || :bottom_right
                            margin = @format_options[:watermark][:pixels_from_edge] || @format_options[:watermark][:margin]
                            margin_width = margin || @format_options[:watermark][:margin_width] || 10
                            margin_height = margin || @format_options[:watermark][:margin_height] || 10
                            positioning = case position
                                          when 'bottom_left'
                                            "#{margin_width}:main_h-overlay_h-#{margin_height}"
                                          when 'bottom_right'
                                            "main_w-overlay_w-#{margin_width}:main_h-overlay_h-#{margin_height}"
                                          when 'top_left'
                                            "#{margin_width}:#{margin_height}"
                                          when 'top_right'
                                            "main_w-overlay_w-#{margin_width}:#{margin_height}"
                                          end

                            ["-vf", "movie=#{path} [logo]; [in][logo] overlay=#{positioning} [out]"]
                          end
  end
end

CarrierWave::Video::FfmpegOptions.prepend(FixWatermarkParams)
```
