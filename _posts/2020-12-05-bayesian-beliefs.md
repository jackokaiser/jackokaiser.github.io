---
title: Bayesian beliefs app
date: 2020-12-05
permalink: /posts/2020/12/bayesian-beliefs
tags:
  - vue.js
  - PWA
---

I have developed a light web app in vue.js using the [quasar framework](https://quasar.dev/).
It is available as [a website](http://jacqueskaiser.com/bayesian_beliefs/#/) as well as an app in the [android play store](https://play.google.com/store/apps/details?id=org.capacitor.bayesianbelief).

This app helps you keep rational beliefs about the world using the Bayesian approach.
It works by listing down competing hypothesis, and maintaining their probability of being true.
This probability is re-calculated with the Bayesian formula everytime a new observation about the world is made.

This is a pure frontend app: all data stays in local storage on the device, there is no backend.
The code is [open-source on github](https://github.com/jackokaiser/bayesian_beliefs).
