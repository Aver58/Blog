---
layout:     post
title:      "图片所占用的内存"
subtitle:   "操作系统入门"
date:       2019-07-04 16:00:00
author:     "Zero"
header-img: "img/post-bg-2015.jpg"
tags:
    - OS
---

# 图片所占用的内存 -- 简单学习

以一张尺寸为900 × 600的图片为例，图片共有像素数：

900 × 600 = 540,000像素(Pixel)。

如果图片是RGB 色彩模式，占用的内存是：

900 × 600 × 3 = 1,620,000 字节(bytes).


图片类型 | 每像素多少字节
---|---
1 比特 数据图(Line art)| 每像素1/8字节，也是一个比特。
8 比特灰度(Grayscale) |每像素1字节。
16 比特灰度(Grayscale)| 每像素2字节。
24 比特 RGB |  每像素3字节，这是图片中最常用的，如JPG格式。
32 比特 印刷色彩模式(CMYK) |每像素4字节
48 比特 RGB | 每像素6字节


1. 如iPhone 6 上设备分辨率大小的RGB类型图片占用内存多少？
> 由于iPhone 6的设计分辨率是750x1334，计算所有像素点 = 750X1334X3 。
> 1. 图片占用内存大小 = (750X1334X3) / (1024X1024) , 也就是 2.86M.

2. 一张6x4寸的图片在150dpi设备上，占用多少内存？
> 1. 首先是计算像素点， 像素点 = (6 x 150) x(4 x 150) = 540,000像素
> 2. 如果图片是RGB类型，则占用内存为(540,000 x 3)/(1024 x 1024) = 1.545M。
