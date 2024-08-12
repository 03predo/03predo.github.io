---
permalink: /video_streaming
layout: page
title: Video Streaming
page_title: OV7670 Video Streaming Server and Client
---

![video_server](assets/imgs/video_server.jpg)

In this project I created a server with the ESP32 that uses an OV7670 to capture images and stream them to a client application over UDP. I created two applications, the server which was responsible for capturing and sending frames upon request from a client, and the client was responsible for making these requests at a given frame rate and displaying them as they appeared. The server runs on the ESP32 and the client runs on an x86 machine running ubuntu. The source code for the [server](https://github.com/03predo/esp32_stream_server) and [client](https://github.com/03predo/stream_client) can be found on github.

# OV7670 Driver
The ESP32 communicates with the OV7670 over two main channels, the SCCB interface and the video port. SCCB is a communication protocol that was created by omnivision but is compatable with I2C so the OV7670 SCCB interface can be connected one of the ESP32 I2C ports. The video port consists of 8 data lines used to transmit the bytes that make up a frame, the HREF, PCLK, and VSYNC lines are used to dictate the timing of this data. The ESP32 I2S interface can be configured to receive frames over this protocol.

![ov7670_block](assets/imgs/ov7670_block.png)

The pixel format we used was RGB565. This means that pixel was represented by 2 bytes, 5 bits for red, 6 bits for green, and 5 bits for blue. We used QVGA (320 x 240) resolution to keep the frame buffer small to avoid stack overflow on the ESP32. To configure the OV7670 with these options control registers are set using the SCCB interface.

![rgb565_timing](assets/imgs/rgb565_timing.jpg)

The above figure shows the timing of a single row of pixels in a frame. When HREF goes high this indicates the start of a row and bytes can be sampled from the data lines on the rising edge of the PCLK. Since we are using RGB565 a single pixel is transmitted in 2 bytes, one after the other. The below figure shows the timing of an entire frame. The VSYNC signal indicates the start and end of a frame.

![vga_frame_timing](assets/imgs/vga_frame_timing.jpg)

With this timing information the ESP32 I2S interface can be configured to receive frames from the OV7670. The figure below shows the hardware connections made between the ESP32 and OV7670. SD corresponds to the 8 data lines from the OV7670 video port.

![esp32_ov7670_connections](assets/imgs/esp32_ov7670_connections.jpg)

Upon sampling the data from the OV7670 the ESP32 then copies these samples into a fifo. This fifo contains 64 elements each of which is 32 bits wide. In camera mode the ESP32 I2S controller copies two samples into a single fifo element, one into the upper 16 bits and the other into the lower 16 bits. Because the OV7670 data bus is only 8 bits wide these samples only take up the lower 8 bits of there respective 16 bit fields. These fifo elements are then copied into main memory via DMA.

![i2s_controller_block](assets/imgs/i2s_controller_block.jpg)

To setup the DMA transfer from the I2S interface to main memory you need to create a linked list of DMA descriptors. These descriptors contain some metadata, a pointer to the buffer in main memory you would like the peripheral data to be copied to, and a pointer to the next descriptor. The first descriptor in the list is given to the I2S controller which will initiate the DMA transfers based on the incoming data. You also supply the I2S controller with a threshold of fifo elements, when the number of fifo elements copied by the DMA reaches this threshold it will trigger an interrupt for the CPU to indicate the elements have been copied.

# Stream Server and Client

The stream server receives a request from a client for a new frame, captures the frame from the ov7670 and sends this frame back to the client via UDP. Because the size of a single QVGA RGB565 frame is 153600 bytes an entire frame cannot be sent in a single UDP transmission, so the frame is sliced into parts each with a corresponding sequence number, sent over udp. The client can then reconstruct the image on there side using the sequence numbers. The reconstructed buffer is then flushed to an OpenGL window.

![stream_server_client](assets/imgs/stream_server_client.jpg)

The video below shows the video stream received by the client from the server on the ESP32. As of right now the frame rate is around 5 FPS, further work needs to be done ensure a more stable connection before increasing the speed.

[![esp32_video_streaming](https://img.youtube.com/vi/NAy5VsRxw_o/0.jpg)](https://www.youtube.com/watch?v=NAy5VsRxw_o)
