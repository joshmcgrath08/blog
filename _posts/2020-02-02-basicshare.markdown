---
layout: post
title:  "Share Password-Protected Secrets Using End-to-End Encryption in a Browser"
tags: react lambda python wsgi privacy
comments: true
---

A couple of years ago I was looking for a solution that would allow me to share password-protected content with my mom. At the time, most of the solutions I found were  clunky (e.g. share a password protected PDF), infeasible (e.g. she would need to have an account on the site as well), or required that I trust them (e.g. handing over my data to the site in a way that they could read it if they chose).

I decided to build a solution that, had I found in my search, I would have used (i.e. easy to use and couldn't read my data). While it's still mostly a proof of concept, I built a small React + Flask on Lambda solution that uses client-side encryption and two communication channels in order to achieve end-to-end encryption.

Because I am pretty lacking in the creativity department, I called the site [BasicShare](https://basicshare.io).

And since I've had some free time recently, I decided to clean it up and put the [source on Github](https://github.com/joshmcgrath08/basicshare).
