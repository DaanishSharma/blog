---
title: I learned sensor fusion from zero and built micronav
date: 2026-11-08
excerpt: Every sensor in your phone lies. An open-source positioning engine in plain numpy, tested on a real drive, that stays 3x closer than a GPS-only filter when GPS drops out.
tags: [sensor fusion, kalman filter, navigation, open source, micronav]
---

![micronav launch video](media/micronav-launch.mp4)

Every map app answers one question before anything else: **where am I?** I wanted to know how that really works, so I learned sensor fusion from zero and built **[micronav](https://github.com/DaanishSharma/micronav)**, an open-source positioning engine in plain numpy.

```bash
pip install micronav
micronav run "my-phyphox-recording.zip" --out map.html
```

## Every sensor lies

Your phone has three sensors that know something about where it is. Each one is wrong in its own way:

| Sensor | What it measures | How it lies (simulated, one minute) |
|---|---|---|
| GPS | position, once a second | jumps by metres, drops out under flyovers |
| Gyroscope | how fast it's turning | a tiny bias: **25° off** after a minute |
| Accelerometer | acceleration, plus gravity | integrate twice: **119 m off** after a minute |

Sensor fusion means combining them so that each one covers for the others' mistakes.

## The test drive

I recorded an 11-minute, 4.3 km drive in Patiala with [phyphox](https://phyphox.org) (also open source) on an iPhone 13. That gave me the accelerometer, gyroscope and magnetometer at 100 Hz, and 698 GPS fixes.

## What's inside, chapter by chapter

1. **The Kalman filter in 1-D.** A guess, plus how unsure you are. With a constant, it turns into the running average.
2. **Orientation.** An error-state Kalman filter: the gyro predicts, gravity fixes tilt, and the compass fixes heading. An NIS gate throws out readings near steel.
3. **GPS smoothing.** A constant-velocity Kalman filter, plus an RTS smoother that lets later fixes correct earlier estimates.
4. **Map matching.** A hidden Markov model with Viterbi on OpenStreetMap roads, so the track doesn't jump between parallel streets.
5. **How the phone sits in the car.** Worked out from braking (GPS speed changes) and turning (GPS speed × gyro). There's a closed-form answer.
6. **One 15-state error-state EKF** for position, velocity, orientation and both sensor biases. The IMU predicts at 100 Hz and GPS corrects at 1 Hz. It also knows **cars don't slide sideways**.
7. **The road as a measurement** inside the filter.
8. **Batch smoothing.** Write everything as one big least-squares problem and you get *exactly* the RTS answer. The test checks they agree to 1e-8.

Every filter is tested against a simulation where the true path is known.

## The result: losing GPS

To test without ground truth, I hid GPS for 30 s out of every 90 and measured how far each filter's guess was from the fixes it never saw:

| | median miss |
|---|---|
| Kalman filter on GPS alone | **29 m** |
| micronav 15-state filter | **9 m** |

With 60 s gaps, it's 55 m against 28 m.

## Things that surprised me

- **The phone's GPS accuracy number is pessimistic.** If the reported accuracy were right, the median NIS would be 1.39. Mine was 0.016. The fixes are already smoothed, so their errors are correlated. Check before you trust a filter's ± uncertainty.
- **"Cars don't slide sideways" is a powerful rule.** In the simulation, it cuts outage drift by about a third on its own.
- **Recursion and batch are the same answer.** The RTS smoother and one big least-squares solve give identical tracks for a linear model.

## What's next

A native phone recorder instead of phyphox, NavIC (India's own satellite navigation), and more open map sources next to OpenStreetMap.

Code: [github.com/DaanishSharma/micronav](https://github.com/DaanishSharma/micronav) (MIT). References: Barfoot, *State Estimation for Robotics*; Solà et al., *A micro Lie theory*; Newson & Krumm (2009) on HMM map matching. Map data © OpenStreetMap contributors.
