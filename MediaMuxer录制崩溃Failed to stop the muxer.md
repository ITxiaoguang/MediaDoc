一些机型报错/闪退/录屏失败

**MediaMuxer 报错 Failed to stop the muxer**

**java.lang.IllegalStateException: Failed to stop the muxer**

```
[MPEG4Writer] : Audio track stopping. Stop source
[MPEG4Writer] : Audio track source stopping
[MPEG4Writer] : Received total/0-length (48/0) buffers and encoded 47 frames. - Video
[MPEG4Writer] : Audio track source stopped
[MPEG4Writer] : Audio track stopped. Stop source
[MPEG4Writer] : Video track stopping. Stop source
[MPEG4Writer] : EIS Audio durationUs(0), Video durationUs(0)
[MPEG4Writer] : Video track source stopping
[MPEG4Writer] : Video track source stopped
[MPEG4Writer] : Video track stopped. Stop source
[MPEG4Writer] : Duration from tracks range is [9333, 4694275] us
[MPEG4Writer] : Stopping writer thread
[MPEG4Writer] : 0 chunks are written in the last batch
[MPEG4Writer] : Writer thread stopped
[MediaCodec] : ~MediaCodec
[MediaCodec] : MediaCodec::updateAnalyticsItem
[System.err] : java.lang.IllegalStateException: Failed to stop the muxer
[System.err] : 	at android.media.MediaMuxer.nativeStop(Native Method)
[System.err] : 	at android.media.MediaMuxer.stop(MediaMuxer.java:466)
[System.err] : 	at com.cylan.webrtc.sdk.VideoFileRenderer.release(VideoFileRenderer.java:166)
[System.err] : 	at com.cylan.webrtc.sdk.CyRtcClient.stopRecord(CyRtcClient.kt:90)
[System.err] : 	at com.cylan.imcam.biz.player.LiveFragment$addViewListener$1$1.invoke(LiveFragment.kt:452)
[System.err] : 	at com.cylan.imcam.biz.player.LiveFragment$addViewListener$1$1.invoke(LiveFragment.kt:437)
[System.err] : 	at com.cylan.imcam.widget.CheckImage.setChecked(CheckImage.kt:82)
[System.err] : 	at com.cylan.imcam.widget.CheckImage.toggle(CheckImage.kt:97)
[System.err] : 	at com.cylan.imcam.widget.CheckImage.performClick(CheckImage.kt:88)
[System.err] : 	at android.view.View.performClickInternal(View.java:7165)
[System.err] : 	at android.view.View.access$3500(View.java:814)
[System.err] : 	at android.view.View$PerformClick.run(View.java:27670)
[System.err] : 	at android.os.Handler.handleCallback(Handler.java:883)
[System.err] : 	at android.os.Handler.dispatchMessage(Handler.java:100)
[System.err] : 	at android.os.Looper.loop(Looper.java:230)
[System.err] : 	at android.app.ActivityThread.main(ActivityThread.java:7991)
[System.err] : 	at java.lang.reflect.Method.invoke(Native Method)
[System.err] : 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:526)
[System.err] : 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1034)
```




**然后在前面发现 Audio 写入 drainAudio 报错：** **writeSampleData returned an error**

定位到是 Audio 写入报错。

注意 presentation time: 0 这个日志，提交的时间一直都是0




```
[MediaCodec] : presentation time: 0
[MediaCodec] : -- empty queue, so ignore that.
[MPEG4Writer] : Condition timestampUs >= 0LL failed for Audio track
[MPEG4Writer] : 0 frames to dump timeStamps in Audio track 
[MediaCodec] : presentation time: 0
[MPEG4Writer] : 0-duration samples found: 1
[MPEG4Writer] : Received total/0-length (2/0) buffers and encoded 1 frames. - Audio
[IM] : VideoFileRendererAudioThread|(VideoFileRenderer.java:330) 音频 记录 线程id：3276
[MPEG4Writer] : Audio track drift time: 0 us
[MediaCodec] : presentation time: 0
[MediaCodec] : -- empty queue, so ignore that.
[MediaAdapter] : pushBuffer called before start
[VideoFileRenderer] : writeSampleData returned an error
  java.lang.IllegalStateException: writeSampleData returned an error
	at android.media.MediaMuxer.nativeWriteSampleData(Native Method)
	at android.media.MediaMuxer.writeSampleData(MediaMuxer.java:694)
	at com.cylan.webrtc.sdk.VideoFileRenderer.drainAudio(VideoFileRenderer.java:329)
	at com.cylan.webrtc.sdk.VideoFileRenderer.$r8$lambda$6RMOZfT1cxU4P14fozUrx1kjd00(Unknown Source:0)
	at com.cylan.webrtc.sdk.VideoFileRenderer$$ExternalSyntheticLambda1.run(Unknown Source:2)
	at android.os.Handler.handleCallback(Handler.java:883)
	at android.os.Handler.dispatchMessage(Handler.java:100)
	at android.os.Looper.loop(Looper.java:230)
	at android.os.HandlerThread.run(HandlerThread.java:67)
```

看代码里的 MediaCodec audioEncoder

```
private MediaCodec audioEncoder;
// 打印 presTime 一直都是 0，疑似这里出了问题。
public void onWebRtcAudioTracking(ByteBuffer byteBuffer, int bytesWrite, boolean b) {
    ...
    audioEncoder.queueInputBuffer(bufferIndex, byteBuffer.arrayOffset(), bytesWrite, presTime, 0);
    presTime += (bytesWrite / (sampleRate * numChannels * 2) * 1000000L);
}
```




网上搜索 **writeSampleData returned an error**

在这里找到答案：

<https://github.com/natario1/Transcoder/issues/137>

```
var presentationTimeUs = 0L

...
private class EosIgnoringDataSink{

    override fun writeTrack(type: TrackType, byteBuffer: ByteBuffer, bufferInfo: MediaCodec.BufferInfo) {
        if (bufferInfo.presentationTimeUs < presentationTimeUs){
            return
        }
        presentationTimeUs = bufferInfo.presentationTimeUs

            ...
    }
```

说明 presentationTimeUs 时间必须递增, 把1000000L往前面提，确保递增。

```
private MediaCodec audioEncoder;

public void onWebRtcAudioTracking(ByteBuffer byteBuffer, int bytesWrite, boolean b) {
    ...
    // 修改这句代码
    presTime += (bytesWrite * 1000000L / (sampleRate * numChannels * 2L));
    audioEncoder.queueInputBuffer(bufferIndex, byteBuffer.arrayOffset(), bytesWrite, presTime, 0);    
}
```

确保音频时间递增后放到不同的机型上运行发现成功！

解决bug。