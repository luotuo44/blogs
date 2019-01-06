做过FFmpeg API开发的，估计对`avformat_find_stream_info`函数一定不会陌生。FFmpeg在`该函数中确定每个流的duration，更准确来说是`estimate_timings`函数，前者会调用后者。

[`estimate_timings`](https://github.com/FFmpeg/FFmpeg/blob/b9d2005ea5d6837917a69bc2b8e98f5695f54e39/libavformat/utils.c#L2846)的实现如下:

```c

static void estimate_timings(AVFormatContext *ic, int64_t old_offset)
{
    int64_t file_size;

    /* get the file size, if possible */
    if (ic->iformat->flags & AVFMT_NOFILE) {
        file_size = 0;
    } else {
        file_size = avio_size(ic->pb);
        file_size = FFMAX(0, file_size);
    }

    if ((!strcmp(ic->iformat->name, "mpeg") ||
         !strcmp(ic->iformat->name, "mpegts")) &&
        file_size && (ic->pb->seekable & AVIO_SEEKABLE_NORMAL)) {
        /* get accurate estimate from the PTSes */
        estimate_timings_from_pts(ic, old_offset);
        ic->duration_estimation_method = AVFMT_DURATION_FROM_PTS;
    } else if (has_duration(ic)) {
        /* at least one component has timings - we use them for all
         * the components */
        fill_all_stream_timings(ic);
        ic->duration_estimation_method = AVFMT_DURATION_FROM_STREAM;
    } else {
        /* less precise: use bitrate info */
        estimate_timings_from_bit_rate(ic);
        ic->duration_estimation_method = AVFMT_DURATION_FROM_BITRATE;
    }
    update_stream_timings(ic);

    {
        int i;
        AVStream av_unused *st;
        for (i = 0; i < ic->nb_streams; i++) {
            st = ic->streams[i];
            av_log(ic, AV_LOG_TRACE, "stream %d: start_time: %0.3f duration: %0.3f\n", i,
                   (double) st->start_time * av_q2d(st->time_base),
                   (double) st->duration   * av_q2d(st->time_base));
        }
        av_log(ic, AV_LOG_TRACE,
                "format: start_time: %0.3f duration: %0.3f bitrate=%"PRId64" kb/s\n",
                (double) ic->start_time / AV_TIME_BASE,
                (double) ic->duration   / AV_TIME_BASE,
                (int64_t)ic->bit_rate / 1000);
    }
}
```
上面函数的最后那个局部程序块负责把各个流时长打印出来，这个输出在我们使用ffmpeg、ffplay和ffprobe个工具时经常看到，类似下面
![tools_print_duration](https://raw.githubusercontent.com/luotuo44/blogs/master/img/ffmpeg/tools_print_duration.jpg)

言归正传，mpegts格式文件每个流的时长确定在上面函数的第一个if语句中，也就是[`estimate_timings_from_pts`](https://github.com/FFmpeg/FFmpeg/blob/b9d2005ea5d6837917a69bc2b8e98f5695f54e39/libavformat/utils.c#L2720)函数，该函数实现如下：

```c

#define DURATION_MAX_READ_SIZE 250000LL
#define DURATION_MAX_RETRY 6

static void estimate_timings_from_pts(AVFormatContext *ic, int64_t old_offset)
{
    AVPacket pkt1, *pkt = &pkt1;
    AVStream *st;
    int num, den, read_size, i, ret;
    int found_duration = 0;
    int is_end;
    int64_t filesize, offset, duration;
    int retry = 0;

    /* flush packet queue */
    flush_packet_queue(ic);

    for (i = 0; i < ic->nb_streams; i++) {
        st = ic->streams[i];
        if (st->start_time == AV_NOPTS_VALUE &&
            st->first_dts == AV_NOPTS_VALUE &&
            st->codecpar->codec_type != AVMEDIA_TYPE_UNKNOWN)
			//留意下面的输出，如果你的ts文件是两个ts文件拼接而成，并且第一个ts文件没有音频流，那么下面的输出就可能会打印出来。
            av_log(ic, AV_LOG_WARNING,
                   "start time for stream %d is not set in estimate_timings_from_pts\n", i);

        if (st->parser) {
            av_parser_close(st->parser);
            st->parser = NULL;
        }
    }

    av_opt_set(ic, "skip_changes", "1", AV_OPT_SEARCH_CHILDREN);
    /* estimate the end time (duration) */
    /* XXX: may need to support wrapping */
    filesize = ic->pb ? avio_size(ic->pb) : 0;
    do {
		//这层循环确定遍历次数，一般只会执行一次。如果拼接一个没有音频的片尾可能会两次
		//找到任意一个流都会将found_duration置为1。
        is_end = found_duration;
        offset = filesize - (DURATION_MAX_READ_SIZE << retry);
        if (offset < 0)
            offset = 0;

		//由于ts文件头没有记录每个流的时长，因此这里根据最后一个packet的pts确定时长。
		//从文件最后filesize-offset字节中解析得到一个个的packet,进而得到pts。流的
		//duration等于显然等于最后一个pts减去该流的第一个packet的pts(当然还要加上
		//最后一个packet的时长)。下面的for循环会不断读取packet，越后面的packet的pts
		//就越大，于是不断刷新duration的值。
        avio_seek(ic->pb, offset, SEEK_SET);
        read_size = 0;
        for (;;) {
            if (read_size >= DURATION_MAX_READ_SIZE << (FFMAX(retry - 1, 0)))
                break;

            do {
                ret = ff_read_packet(ic, pkt);
            } while (ret == AVERROR(EAGAIN));
            if (ret != 0)
                break;
            read_size += pkt->size;
            st         = ic->streams[pkt->stream_index];//由packet确定所属的流
            if (pkt->pts != AV_NOPTS_VALUE &&
                (st->start_time != AV_NOPTS_VALUE || st->first_dts  != AV_NOPTS_VALUE)) {
                if (pkt->duration == 0) {
                    ff_compute_frame_duration(ic, &num, &den, st, st->parser, pkt);
                    if (den && num) {
                        pkt->duration = av_rescale_rnd(1,
                                           num * (int64_t) st->time_base.den,
                                           den * (int64_t) st->time_base.num,
                                           AV_ROUND_DOWN);
                    }
                }
                duration = pkt->pts + pkt->duration;
                found_duration = 1;
                if (st->start_time != AV_NOPTS_VALUE)
                    duration -= st->start_time;
                else
                    duration -= st->first_dts;
                if (duration > 0) {
                    if (st->duration == AV_NOPTS_VALUE || st->info->last_duration<= 0 ||
                        (st->duration < duration && FFABS(duration - st->info->last_duration) < 60LL*st->time_base.den / st->time_base.num))
                        st->duration = duration;
                    st->info->last_duration = duration;
                }
            }
            av_packet_unref(pkt);
        }

        /* check if all audio/video streams have valid duration */
        if (!is_end) {
            is_end = 1;
            for (i = 0; i < ic->nb_streams; i++) {
                st = ic->streams[i];
				//上面的for(;;)循环中找齐音视频的时长后，is_end就保持为1，在后面的
				//while中退出。
                switch (st->codecpar->codec_type) {
                    case AVMEDIA_TYPE_VIDEO:
                    case AVMEDIA_TYPE_AUDIO:
                        if (st->duration == AV_NOPTS_VALUE)
                            is_end = 0;
                }
            }
        }
    } while (!is_end &&
             offset &&
             ++retry <= DURATION_MAX_RETRY);

    av_opt_set(ic, "skip_changes", "0", AV_OPT_SEARCH_CHILDREN);

    /* warn about audio/video streams which duration could not be estimated */
    for (i = 0; i < ic->nb_streams; i++) {
        st = ic->streams[i];
        if (st->duration == AV_NOPTS_VALUE) {
            switch (st->codecpar->codec_type) {
            case AVMEDIA_TYPE_VIDEO:
            case AVMEDIA_TYPE_AUDIO:
                if (st->start_time != AV_NOPTS_VALUE || st->first_dts  != AV_NOPTS_VALUE) {
                    av_log(ic, AV_LOG_DEBUG, "stream %d : no PTS found at end of file, duration not set\n", i);
                } else
                    av_log(ic, AV_LOG_DEBUG, "stream %d : no TS found at start of file, duration not set\n", i);
            }
        }
    }
    fill_all_stream_timings(ic);

    avio_seek(ic->pb, old_offset, SEEK_SET);
    for (i = 0; i < ic->nb_streams; i++) {
        int j;

        st              = ic->streams[i];
        st->cur_dts     = st->first_dts;
        st->last_IP_pts = AV_NOPTS_VALUE;
        st->last_dts_for_order_check = AV_NOPTS_VALUE;
        for (j = 0; j < MAX_REORDER_DELAY + 1; j++)
            st->pts_buffer[j] = AV_NOPTS_VALUE;
    }
}
```

上面代码的解释了该函数是如何找到每一个流的duration，但有一点要注意：上面代码是在一个if判断体里面设置每一个流的duration。判断的条件是`(st->start_time != AV_NOPTS_VALUE || st->first_dts  != AV_NOPTS_VALUE)`，在上述代码的一开始我就在注释中提及要注意输出，因为这个输出的打印条件就是这个判断条件。
如果ts文件确实在前面拼接了另外一个没有音频的文件，那么上面的do{}while()循环并没有设置音频流的duration。那在哪里能设置呢？答案是[`fill_all_stream_timings`](https://github.com/FFmpeg/FFmpeg/blob/b9d2005ea5d6837917a69bc2b8e98f5695f54e39/libavformat/utils.c#L2644)函数，其实现如下：

```c
static void fill_all_stream_timings(AVFormatContext *ic)
{
    int i;
    AVStream *st;

    update_stream_timings(ic);
    for (i = 0; i < ic->nb_streams; i++) {
        st = ic->streams[i];
        if (st->start_time == AV_NOPTS_VALUE) {
            if (ic->start_time != AV_NOPTS_VALUE)
                st->start_time = av_rescale_q(ic->start_time, AV_TIME_BASE_Q,
                                              st->time_base);
            if (ic->duration != AV_NOPTS_VALUE)
                st->duration = av_rescale_q(ic->duration, AV_TIME_BASE_Q,
                                            st->time_base);
        }
    }
}
```
相当简单粗暴，直接将AVFormatContext(也就是文件本身)的start_time和duration设置成该流的start_time和duration。这会使得音频duration设置得比实际的要大。


是不是在ts文件头拼接另外一个音频流的文件就一定会使得音频流的start_time为`AV_NOPTS_VALUE`呢？并不是这样的。根据[雷神](http://blog.csdn.net/leixiaohua1020/article/details/44084321#t4)的博文，FFmpeg会尝试预读多媒体文件前面的若干帧。如果若干帧包含音频帧，start_time就不为`AV_NOPTS_VALUE`。也就是说：如果拼接的文件头比较短，那么FFmpeg能预读取到前面第一个音频帧，FFmpeg能正确计算音频流的duration；如果拼接的文件头比较长，那么FFmpeg不能正确计算音频流的duration。

还有一个[`update_stream_timings`](https://github.com/FFmpeg/FFmpeg/blob/b9d2005ea5d6837917a69bc2b8e98f5695f54e39/libavformat/utils.c#L2559)函数需要提一下。这个有点长就不贴代码了。它的主要功能是计算每一个流的时长，然后将最大的时长赋值给AVFormatContext的duration，然后根据这个时长和文件大小计算文件的码率。
