---
title: Developer Guides
weight: 10
aliases:
    - ../os/developer-guides
---

Most users will never have to build Flatcar Container Linux from source or modify it in any way. If you have a need to modify Flatcar Container Linux, we provide an SDK that allows you to build your own developer images. We also provide OEM functionality for cloud providers and other companies that must customize Flatcar Container Linux to run within their environment.

* [Guide to building custom Flatcar images from source][mod-cl]
* [Building production images][production-images]
* [Building custom kernel modules][kernel-modules]
* [SDK tips and tricks][sdk-tips]
* [SDK build process][sdk-bootstrapping]
* [Disk layout][disk-layout]
* [Kola integration testing framework][mantle-utils]

[sdk-tips]: sdk-tips-and-tricks
[disk-layout]: sdk-disk-partitions
[production-images]: sdk-building-production-images
[mod-cl]: sdk-modifying-flatcar
[kernel-modules]: kernel-modules
[sdk-bootstrapping]: sdk-bootstrapping
[mantle-utils]: https://github.com/kinvolk/mantle/blob/flatcar-master/README.md#kola
