---
layout: article
title: "Neuromancer"
modified:
categories:
excerpt: "A brain-computer interface that measures human attention by
reading minds. Neuroscience + machine learning = magic"
tags: [neuroscience]
image:
  feature: joe_eeg_feature.jpg
  teaser: joe_eeg_teaser.png
  thumb:
date: 2014-11-28T00:52:18-05:00
---

Our brains often misbehave; they make us bored, anxious, procrastinatory,
they get distracted at all the wrong times. Many of us accept this behavior as
an unchangeable part of life, and why not? Thought is a difficult process to
debug, given what limited access we have to our true internal mental state. What
we need to overcome our mental bad habits, then, is a sort of debugger; a device
that tells us more than what we can intuit given our own stream of consciousness.

Neuromancer is such a device for helping us improve our concentration. Comprised
of an EEG headset (either an [Emotiv](www.emotiv.com) or BioSemi EEG) and a software
suite, it aims to improve our focus by giving us real-time neurofeedback on our
current state of concentration. The Neuromancer package, whose (alpha) source can be found
[here](https://github.com/dalequark/neuromancer), consists of a Java backend,
browser frontend, and a handful of Python scripts for data processing.  Neuromancer
provides users with a concentration-based task while simultaneously measuring neural
signals via EEG. After training a classifier that determines the relative face/place
focus of a user, Neuromancer gives users real time neurofeedback on their focus
by altering the presented stimulus.  fMRI studies have shown that such an experiment
can improve participants' performance on concentration tasks, even after the
neurofeedback is removed, as compared to controls.  

More details to come soon. Stay tuned!
