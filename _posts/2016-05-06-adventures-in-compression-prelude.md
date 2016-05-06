---
layout: post
title: "Adventures in compression: prelude"
description: ""
category: compression learning
tags: []
---
{% include JB/setup %}

Compression is one of those things a lot of us take for granted.  Shove in bits, get less bits out.  Take those bits, shove them into something else, get the original bits back.  _Magic!_

This magic is so commonplace, though, that we hardly ever notice it, and for me, sometimes don't fully understand how it works.

If we sat and thought about it for a while, we'd come up with educated guesses, and on the face of it, it's simple: take one thing, replace it with a smaller thing that we can figure out how to convert back to a big thing.  There's actually a wide range of approaches to doing this, though.  They all have their individual trade-offs when it comes to the speed of compression and decompression, the CPU or memory usage, or the compression ratio.  Then there's the data being compressed, which itself can change the effectiveness of a given compression algorithm.  There's a lot going on.

Working on a side project, I came across a library that supports decompression for a lot of algorithms, but no compression!  I want the whole enchilada!  Thus, I started reading about [DEFLATE](https://en.wikipedia.org/wiki/DEFLATE), and [Lempel-Ziv-Welch](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch), and [Shannon-Fano coding](https://en.wikipedia.org/wiki/Shannon%E2%80%93Fano_coding), and, well... it's a lot to digest!

My intent is to write about my experience of learning, and hopefully, understanding, these building blocks of compression algorithms, and hopefully, be able to sufficiently explain them to others who aren't familiar.

Stay tuned!
