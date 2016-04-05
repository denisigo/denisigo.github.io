---
layout: post
title: Creating Network Radio Player on Android
category: Android
---
Hi, today I'm gonna show you how to create a simple Android network radio client which you can use to listen to your favorite station - just type in one's URL! For the sake of simplicity, I'll try to divide this topic into several articles starting from the simplest version - v1. This version will only support plain stream (Shoutcast, Icecast) of the "audio/mpeg" type and will have minimal sufficient amount of code. I'll try to evolve this app during the time and write about it here. As always, check out the code on [GitHub](https://github.com/denisigo/NetRadioPlayer/tree/v1). 

In this article you'll learn how to use HTTPUrlConnection for connection to radio host, InputStream to read stream data, native [mpg123 decoder](http://www.mpg123.de/) to decode MPEG data to PCM, JNI lib to access native decoder from Java, and AudioTrack to play decoded PCM data.

<!--more-->

# Before we start

You should already be familiar with the Android SDK and have installed [Android NDK](http://developer.android.com/tools/sdk/ndk/index.html) which we will need to compile native code. You better also be familiar with the C, C++ and [JNI](http://developer.android.com/training/articles/perf-jni.html) to better understand what's happening. And lastly, I'm using Ubuntu Linux, so you're free to repeat the same on another operating systems.

# How it will work

Here is the image to illustrate whole process: ![](https://docs.google.com/drawings/d/1FXxP3TW9OnjycD_jQ0SfkWeXYVXp-LfBfFG0R4RZclI/pub?w=923&h=652)
