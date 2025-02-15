---
type: "post"
title: Caffe with Python3 on Centos 7 Install Notes
---

I've posted a gist [here](https://gist.github.com/graphaelli/7a104545be9e288d94bc) with a Dockerfile for building a
Centos 7 + Python 3 + caffe container image.  I don't expect many will use this directly but the steps and various
patches might be useful for incorporation into your own Dockerfiles.  As such, I've put no effort into minimizing the
size or cacheability of layers.  `docker build` as-is weighs in at 2.4+ GB.

The default docker run expects a `/notebooks` volume and exposes jupyter-notebook on port 8888.
