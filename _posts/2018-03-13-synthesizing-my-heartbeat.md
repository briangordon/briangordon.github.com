---
layout: post
title: Synthesizing my heartbeat
tags: programming math
published: false
---

I recently got my hands on a nice, hospital-grade pulse oximeter for analyzing my breathing patterns while I sleep. You can view the data one sample at a time on the device, but what's really interesting is that you can transfer the data to your computer over USB. Let's see what we can do with it!

- putty screenshots
- create a sound clip of a heartbeat in realtime
- histogram of spo2 and plot % time spent under each point
- frequency analysis of spo2 dip events
- synthesize a speeded-up sound clip in the frequency-time domain where the frequency equals the spo2 at that time
