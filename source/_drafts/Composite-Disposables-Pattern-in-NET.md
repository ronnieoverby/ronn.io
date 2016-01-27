title: Composite Disposables Pattern in .NET
tags:
  - .net
  - patterns
categories:
  - Code
date: 2016-01-27 10:34:00
---
# outline
0. problem with disposables that contain other disposables
   - once you need to pass disposable to other scope, proper disposal can get tricky
   - some implementations take a bool flag to dispose dependency (mostly streams) but not unbiquitous in framework
0. solved by building a CompositeDisposable class
0. new IDisposable implementations can take a params IDisposable[] in their constructor
   - params turned into CompositeDisposable and stored
   - when Disposed, composited disposable is disposed, too
0. is it too late for this pattern? the framework is already built
   - explore ways to adapt existing apis
   
   
