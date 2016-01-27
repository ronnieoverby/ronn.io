title: A Simpler CryptoStream
tags:
  - crypto
  - .net
categories:
  - Code
date: 2016-01-26 20:39:00
---
# Outline
0. Background about that led to enc/dec scenario
0. The problem with the current implementation (what my code looked like)
   - createEnc/createDec/disposables everywhere
   - reuse
0. Wrapping underlying stream/cryptostreams/cryptoTransforms
0. Code after

### Background

I'm working on an application that has to transfer sensitive financial data over public networks. These files can get to be pretty large (hundreds of megabytes) and it's possible that the financial institutions using the system will send many of them at once.

When sending files, it's nice to show the user a {% post_link Hacking-Event-Sequences-with-Reactive-Extensions Progress Bar™ %}. But, in order to know how much further you've left to go, you gotta know where you're going. In terms of sending a file somewhere, the system needs to know how many bytes are being sent. That means that I have to generate the file in it's entirety before beginning to stream the data over the network. I guess I could build the file on one thread, begin sending the data on anothers, and work my way up to having an accurate progress bar somewhere in the middle of the transfer… but dammit if this system isn't already complicated enough.

Okay, geez, enough about progress bars. Another property of the system is that it has a few dependencies that are forcing me to build in a 32-bit architecture. And that means that I have to be careful when I'm building these files. In my experimentation I put a mid-sized workload to the test and sure enough, I encountered an `OutOfMemoryException`. Crap. I have to save the data to a temporary file on disk and then stream the data from the disk to the network.

Now I've got *another* problem: the data is sensitive. Should any part of this Rube Goldberg machine barf on itself, that temp-file could be left on the disk. Even if I'm diligent about error handling and do everything possible to delete the file, experience tells me that under some dumb circumstances, the program won't have permission to delete it's own temp-file or maybe it will crash before it has the chance to try. What



{% asset_img notsimply.png "It is folly!" %}
