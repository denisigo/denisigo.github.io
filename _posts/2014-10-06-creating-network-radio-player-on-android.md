---
layout: post
title: Creating Network Radio Player on Android
category: Android
---
Hi, today I'm gonna show you how to create a simple Android network radio client which you can use to listen to your favorite station - just type in one's URL! For the sake of simplicity, I'll try to divide this topic into several articles starting from the simplest version - v1. This version will only support plain stream (Shoutcast, Icecast) of the "audio/mpeg" type and will have minimal sufficient amount of code. I'll try to evolve this app during the time and write about it here. As always, check out the code on [GitHub](https://github.com/denisigo/NetRadioPlayer/tree/v1). 

<!--more-->

In this article you'll learn how to use HTTPUrlConnection for connection to radio host, InputStream to read stream data, native [mpg123 decoder](http://www.mpg123.de/) to decode MPEG data to PCM, JNI lib to access native decoder from Java, and AudioTrack to play decoded PCM data.

# Before we start

You should already be familiar with the Android SDK and have installed [Android NDK](http://developer.android.com/tools/sdk/ndk/index.html) which we will need to compile native code. You better also be familiar with the C, C++ and [JNI](http://developer.android.com/training/articles/perf-jni.html) to better understand what's happening. And lastly, I'm using Ubuntu Linux, so you're free to repeat the same on another operating systems.

# How it will work

Here is the image to illustrate whole process: ![](https://docs.google.com/drawings/d/1FXxP3TW9OnjycD_jQ0SfkWeXYVXp-LfBfFG0R4RZclI/pub?w=923&h=652)

1.  When user clicks "Play" on the [MainActivity](https://github.com/denisigo/NetRadioPlayer/blob/v1/app/src/main/java/com/denisigo/netradioplayer/MainActivity.java), create a [StreamPlayer](https://github.com/denisigo/NetRadioPlayer/blob/v1/app/src/main/java/com/denisigo/netradioplayer/StreamPlayer.java) thread instance and provide it with the station URL to play.
2.  In StreamPlayer open [HttpUrlConnection](http://developer.android.com/reference/java/net/HttpURLConnection.html) and get [InputStream](http://developer.android.com/reference/java/io/InputStream.html) from it
    1.  In the while loop read a chunk of [mp3-encoded](https://en.wikipedia.org/wiki/MP3) data from InputStream
    2.  Pass that chunk to the [JNI decoder lib](https://github.com/denisigo/NetRadioPlayer/tree/v1/app/jni/libdecoder-jni)
    3.  Then in turn, pass chunk to the [mpg123 decoder lib](https://github.com/denisigo/NetRadioPlayer/tree/v1/app/jni/libmpg123)
    4.  If mpg123 has decoded some [PCM](https://en.wikipedia.org/wiki/Pulse-code_modulation) data, return it to the StreamPlayer
    5.  Write decoded PCM buffer to the [AudioTrack](http://developer.android.com/reference/android/media/AudioTrack.html) for playback

Let's also illustrate the proccess of decoding in more details for better understanding: ![](https://docs.google.com/drawings/d/1hAxZ58vugvi-rJ9cSwH4Lf9Ddmz3LqubOUSi_QLIIm4/pub?w=932&h=825)

1.  Chunk of data is read from InputStream
2.  Chunk is passed to the java Decoder decode() method
3.  Chunk is passed to the JNI libdecoder decodeNative() method
4.  Chunk is passed to the mpg123 library mpg123_decode() method. It returns status of decoding and further processing depends on it:
    *   MPG123_NEED_MORE - mpg123 needs more data to be able to decode something.
    *   MPG123_NEW_FORMAT - it means mpg123 faced a new format in the stream. In our case it means that there was sufficient data provided to determine stream's params (sample rate, number of channels, bits per sample rate), and we can notify Java Decoder about it to initialize AudioTrack.
        *   Java Decoder's onNewFormatCallback() callback method is invoked and stream's params are set.
    *   MPG123_OK - it means mpg123 has successfully decoded some amount of audio data.
        *   Write decoded data to the AudioTrack.
5.  Return to the step 1.

Quite simple, isn't it? Let's code!

# Prepare mpg123 library for using on Android

Beautiful library [mpg123](http://www.mpg123.de/) will help us to decode MP3 data coming from station stream to PCM data suitable for AudioTrack. Since it's written in C, we have to perform some special operations to make it work on Android. You can find already adapted version in the [repo](https://github.com/denisigo/NetRadioPlayer/tree/v1/app/jni/libmpg123). At first, let's download the library [here](http://www.mpg123.de/download.shtml) and unpack it somewhere. Next, we have to configure it, so go into that folder and run:

{% highlight shell %}
./configure
{% endhighlight %}

You don't need to run _make_ and _make install_ since compiling will be performed by the NDK. Since mpg123 library is not just a decoder, we have to extract only things we really need. So go to the _[mpg123]/src_ folder and locate _libmpg123_ folder - it's decoder itself and that's what we only need. There are some shared files libmpg123 is using - let's just copy them into _libmpg123_ folder to optimize structure. These files are (assuming we're in the _[mpg123]/src_ folder):

*   config.h
*   compat/compat.h
*   compat/compat_impl.h

Next, in order to successfull compiling, we have to change some lines. 

_compat.c_:
{% highlight shell %}
#include "compat/compat_impl.h"
to
#include "compat_impl.h"
{% endhighlight %}

_mpg123.h_:
{% highlight shell %}
#include <fmt123.h>
to
#include "fmt123.h"
{% endhighlight %}

Done, now let's copy libmpg123 folder to our project's JNI folder: _NetRadioPlayer/app/jni/libmpg123/libmpg123_. Now it's time to create [Android.mk](https://github.com/denisigo/NetRadioPlayer/blob/v1/app/jni/libmpg123/Android.mk) file for libmpg123, which is needed for NDK to build our lib. I won't dive into it, just say that our lib will be built as a static one to be linked from our JNI decoder lib. libpmg123 is ready to be built, but let's create another library, which methods we will invoke from Java level through JNI and which will invoke libmpg123 methods in turn - _libdecoder-jni_.

# Create JNI decoder library

Well, our JNI decoder library will consist of only one C file - [libdecoder-jni.c](https://github.com/denisigo/NetRadioPlayer/blob/v1/app/jni/libdecoder-jni/libdecoder-jni.c). It is used as a proxy between Java and mpg123 library and exposes 3 methods named following JNI convention - Java_[package_name]_[class_name]_[method_name](), which will be invoked from Java Decoder class:

*   _jint Java_com_denisigo_netradioplayer_Decoder_initNative()_ - used to initialize both JNI decoder library and mpg123 decoder library. In case of success, it will return non-zero mpg123's decoder handle to be able to talk to it later.
*   _Java_com_denisigo_netradioplayer_Decoder_closeNative(jint handle)_ - used to close both JNI decoder library and mpg123 decoder library and free resources. It uses mpg123's decoder handle to close.
*   _Java_com_denisigo_netradioplayer_Decoder_decodeNative(jint handle, jbyteArray in_buffer, jint size, jbyteArray out_buffer, jint max_size)_ - main method used for decoding _size_ bytes of encoded data from _in_buffer_ and place _max_size_ of decoded data to _out_buffer_.

Let's briefly look at the code. 

**__initNative()_**

{% highlight shell %}
// Init mpg123 decoder
mpg123_init();
// Get mpg123 decoder handle. If hnd is not NULL, everything is OK
mpg123_handle* hnd = mpg123_new(NULL, &ret);
// Set mpg123 to feed mode since we're providing 
// data ourseves and don't want mpg123 to read it for us
ret = mpg123_open_feed(hnd);
{% endhighlight %}

{% highlight shell %}
// Get callback method from Java Decoder class to invoke it 
// when mpg123 decoder faces a new format in the stream
jclass thisClass = (*env)->GetObjectClass(env, thiz);
method_onNewFormatCallback = (*env)->GetMethodID(env, thisClass, 
    "onNewFormatCallback", "(III)V");
{% endhighlight %}

**__decodeNative()_**

{% highlight shell %}
size_t bytes_decoded;
// Main method to decode provided data. It returns status of operation 
// and bytes actually decoded in bytes_decoded if any
ret = mpg123_decode(hnd, in_buf, size, out_buf, max_size, &bytes_decoded);
{% endhighlight %}

# Create Java Decoder class

As were already mentioned above, Java Decoder acts as a proxy between libdecoder and libmpg123 and main StreamPlayer class. It invokes JNI methods of libdecoder and thus follows conventions of writing JNI code. There is seriously nothing interesing, you can inspect this tiny [class](https://github.com/denisigo/NetRadioPlayer/blob/v1/app/src/main/java/com/denisigo/netradioplayer/Decoder.java) yourself.

# Create main StreamPlayer class

This class is serving as main component which reads data from stream, decodes it, and writes it to the AudioTrack. There is also no magic, but let's notice some pieces of code:

{% highlight java %}
// At first try to establish a HTTPUrlConnection to the host
mConnection = (HttpURLConnection) mURL.openConnection();
responseCode = mConnection.getResponseCode();
responseMessage = mConnection.getResponseMessage();
{% endhighlight %}

{% highlight java %}
// Currently we support only audio/mpeg format
String contentType = mConnection.getContentType();
if (!contentType.equals("audio/mpeg")){
    onError("Unsupported content type: " + contentType);
    return;
}
{% endhighlight %}

{% highlight java %}
// Create Buffered input stream to optimise stream reads. BufferedInputStream attempts to read
// as many data as possible in source stream even if you're reading single byte.
mInputStream = new BufferedInputStream(mConnection.getInputStream(), IN_BUFFER_SIZE * 10);
{% endhighlight %}

{% highlight java %}
while (!mIsInterrupted){
    // Read data from mInputStream to inBuffer
    bytesRead = mInputStream.read(inBuffer, 0, IN_BUFFER_SIZE);
    // Although Java docs say that number provided by available() method is not accurate, let's just use it for    estimate amount of data available for reading
    bytesCached = mInputStream.available();

    // We can get less bytes than we actually want, handle all the cases
    // If we didn't received any data, it means we might need some time to wait
    if (bytesRead == 0){
        continue;
    } 
    // If decoder returned negative number, it means error occurred
    else if (bytesDecoded < 0){
        onError("Error while decoding the stream: " + bytesDecoded);
        return;
    }
    // Elsewhere we have decoded some real audio data, write it to audio track
    else {
        // If StreamPlayer just started we need to init AudioTrack.
        // We can't create it before since we have
        // correct stream params only after we decoded some data
        if (mAudioTrack == null) {
            // Check if we if have supported stream
            if (mDecoder.getBitsPerSample() != -1) {
                initAudioTrack(mDecoder.getRate(), mDecoder.getChannels(),
                    mDecoder.getBitsPerSample());
                mAudioTrack.play();
            } else {
                onError("Stream has invalid bits per sample");
                return;
            }
        } else {
            // Finally write PCM buffer to AudioTrack. This call will block until
            // all the data is written
            mAudioTrack.write(outBuffer, 0, bytesDecoded);
        }
    }
}
{% endhighlight %}

Please go to [GitHub](https://github.com/denisigo/NetRadioPlayer/blob/v1/app/src/main/java/com/denisigo/netradioplayer/StreamPlayer.java) for the full source code. Please note, that in this version of app, we don't use any prebuffering mechanism so if your internet connection is not good enough, you might face interruptions in playback. That's all for this time. Stay tuned!

