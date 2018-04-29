---
layout: post
title: Synthesizing my heartbeat
tags: health programming math
published: false
---

I recently got my hands on a hospital-grade [pulse oximeter](https://en.wikipedia.org/wiki/Pulse_oximetry) for analyzing my breathing patterns while I sleep. One can view the data a single sample at a time on the device's display, but what's really interesting is that one can transfer the data to a computer over USB for analysis. Let's see what we can do with it!

After installing a USB->COM bridge driver I was able to establish a serial connection to the device with PuTTY and download all of the data stored in its memory.

![A screenshot of PuTTY streaming the data](/images/putty.png)

This is what the data looks like:

```
TIME                 %SPO2   BPM     PA   Status 
18-FEB-09 10:00:21    97      98     092         
18-FEB-09 10:00:22    97      99     076         
18-FEB-09 10:00:23    97      99     076         
18-FEB-09 10:00:24    97     100     107         
18-FEB-09 10:00:25    97     101     107         
18-FEB-09 10:00:26    97     103     080         
18-FEB-09 10:00:27    97     104     086         
18-FEB-09 10:00:28    98     105     057    MO     
```

The **%SPO2** column indicates my peripheral capillary oxygen saturation, i.e. the saturation of the blood in my finger under the sensor, on a scale from 0 to 100. The **BPM** column indicates my heart rate in beats per minute. The **PA** column indicates my pulse amplitude, i.e. the strength of my heartbeat, which is affected by various factors like arterial wall elasticity and respiration. The **Status** column indicates status events like the sensor becoming disconnected. A motion event in which my body movement disrupted the sensor is recorded at 10:00:28 in the above log.

Let's fire up MATLAB and import a few hours of data. If you haven't used it before, MATLAB has a nice data import tool which can generate code to import your file with options that you select graphically:

![A screenshot of MATLAB's data importer](/images/matlab-importer.png)

Right away let's get a visualization of the distribution of my SpO2 data by plotting the data points against the percentile:

```
import_mar13_data
[spo2percentil,x] = ecdf(oximeterdatamar13.SPO2);
stairs(x, 100*spo2percentil)
```

![Distribution of Spo2](/images/matlab-spo2-distribution.png)

- create a sound clip of a heartbeat in realtime (with pulse amplitude as volume?). interpolate data points (10x?) in case a heartbeat starts halfway between data points
- frequency analysis of spo2 dip events
- synthesize a speeded-up sound clip in the frequency-time domain where the frequency equals the spo2 at that time
