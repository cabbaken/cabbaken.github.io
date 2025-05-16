---
title: AE相关基础知识
date: 2025-03-11T09:46:48Z
lastmod: 2025-03-11T09:46:48Z
categories: ["Camera"]
tags: ["Camera", "AE", "Android"]
---


## 基础概念

### 帧

- 一帧就是一幅图像

### pclk

- 是控制像素输出的时钟，即pixel采样时钟，一个clk采集一个像素点 , 单位MHz。表示是每个单位时间内(每秒)采样的pixel数量。

### 帧率(fps)

- line_length = pclk * line_time
- fps = pclk / (VTS * HTS) = pclk / (frame_length * line_length) = 1 / (frame_length * line_time)
- fps = 1 / (time_per_line\[sec\] * (frame_length))
  - fps即表示1秒内帧数，此公式中line_time单位是秒
  - line_time一组setting只有一个值， 一般是不变的，可看做做常数，调节帧率一般都会通过调整VTS来完成(也就是调整V_Blank，增加帧与帧间隔的时长，自然每秒内能处理的帧数就少)，写frame_length寄存器，

### H_Blank

- 行消隐或称水平消隐，假定曝光起始位置在图像的左上角，对于逐行曝光的 sensor 来说，曝光从第一个像素开始，依次曝光直至这行的最后一个像素曝光结束，这时曝光位置要从此行的尾部快速移动到下一行的头部，开始下一行的曝光，这段行与行之间的返回过程称为H_Blank。
- dummy pixels

### V_Blank

- 场消隐或称垂直消隐，假定曝光起始位置在图像的左上角，曝光完成一帧图像后，曝光位置要从图像的右下角返回左上角，开始新一帧的曝光，这一段时间间隔称为V_Blank。
- dummy line
  ![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241106165153-20250311094648-z9mwn3q.png)

### dummy line

- 虚拟行，用来填充V_blank的行，一般最大曝光行数是要大于有效像素的长度，原因就是dummy line的存在

### frame_offset

- 指最小的dummy_line, 指一帧曝光结束到下次准备好重新开始曝光的时间
- 最大曝光行= VTS - frame_offset(最小曝光行不是frame_offset, datasheet中定义一个最小曝光行，有的低端sensor没有说明)

### line length

- 行长(HTS): 一行的长度，包含H_blank
- line_length = width_number_of_effective_cloumns + H_blank
- dvp：Hsync
- hts = pclk/fps/vts = line_time * pclk

### frame length

- 帧长(VTS)：一帧的行数，包含V_blank
- VTS = frame_length = height_number_of_effective_rows + V_blank(dummy_line) = expouse_line + dummy_line
- VTS >= height_number_of_effective_rows + frame_offset(expouse_line + frame_offset)
- min_integration_time(min_shutter) <= integration_time(shutter) <= VTS - frame_offset
- 曝光行小于帧长(vblank导致)
- dvp: Vsync

### line(row) time

- 曝光一行的时间
- line_time = line_length / pclk (单位：us)
- 曝光一行所用的时间，等于一行的长度除以1秒时间内采样的像素数 (路程/速度 = 时间)

```text
SC/GC:	一行时间 = 1/（帧率*帧长）---> fps = 1000000us/(row_time*vts)

//15fps --> 66.67ms
1帧的时间 / 行时间 = 行长
```

### integration time

- 积分时间，单位为行(H), 通常也称为曝光行，对于逐行曝光(一次曝光一行)的sensor，积分时间是指这一帧曝光了多少行，这是一个相对时间。
- 最大曝光行：最大曝光行= VTS - frame_offset(最小的dummy line)
- texp = (一帧的曝光用时)exp_time_us * PCLK / LINE_LENGTH_CLK

### exposure time

- 曝光时间，指一帧曝光了多长时间，这里是绝对时间 ，单位为(s, ms, us)
- 积分时间是指曝光一帧所用的行数，那么一帧的绝对曝光时间就等于曝光所用行数乘以曝光一行所用的时间
- exposue_time = integration_time * line_time
- exposue_time = integration_time * line_length / pclk
- exposue_time = exposure_line * line_time
  - line_time一般是常数，调节exposure_time曝光时间是通过写exposure_line寄存器实现的(写曝光);
  - 曝光时间以行长为单位
  - 一般要求曝光时间是10ms的整数倍，不然会有flicker现象。原因交流电的频率是50Hz, 完整的正弦波周期时间20ms, 所以曝光时间是10ms的整数倍

### 曝光方式

- a. 滚动快门(电子卷帘快门 electronic rolling shutter)：逐行曝光

  ```text
  一帧图像曝光时间是11ms
  一帧图像用积分时间11行完成了11ms的曝光，假设1H曝光时间是1ms
  一帧图像曝光11ms，一帧内所有的像素曝光了11ms
  ```

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241106165346-20250311094648-qsl08ta.png)
sensor逐行曝光从第一行开始曝光，一个行周期结束之后第二行才开始曝光。依次类推，经过N-1行后第N行开始曝光，第一行曝光结束后开始读出数据，读出一行需要一行周期时间(含行消隐时间，即H_Blank)至第一行完全读出后，第二行刚好开始读出，依次类推，当第N-1 行读完后，第N 行开始读出，直到整幅图像完全读出。

- b.全局快门(global shutter/snapshot shutter): 全局曝光
  全局曝光Sensor的所有行同时开始曝光，并同时结束曝光，在曝光结束后，
  Sensor将所有电子从感光区转到存储区，之后逐行地读出像素数据。
  这样曝光的好处是获得图像每一行的曝光时间比较一致, 并且在拍摄运动物体时图像不会出现偏移和歪斜。

### HDR(High Dynamic Range)

- DOL(digital overlap)模式的HDR的工作原理

  - DOL2： sensor完成一帧曝光，sensor生成2幅图像，图像传输到ISP，ISP再将两幅图像合成为一帧图像
  - DOL3： sensor完成一帧曝光，sensor生成3幅图像，图像传输到ISP，ISP再将三幅图像合成为一帧图像

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241106165730-20250311094648-14itc24.png)

### 曝光行影响

一行的时间有没有影响，看环境和相机，在高亮环境，照度超级高的场景，会局部过曝
但是如果短曝的情况，差一行，可能亮度也差很多
如果实际下的曝光行是7行，但是最终写入shutter 的时候只有6行了，这个时候就不对了，少的那一行曝光，就是通过Dgain_ratio补偿的
exposure_time = exp_line * line_length/pclk
