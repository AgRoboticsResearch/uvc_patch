# uvc_patch for multiple usb cameras

## Quick Start for Kernel Version `5.15.136-tegra`
If you are using the `5.15.136-tegra` kernel, you can directly replace the `uvcvideo.ko` module without rebuilding the kernel:
1. Locate the current `uvcvideo.ko` module:
   ```bash
   modinfo uvcvideo | grep filename
   ```
2. Backup the existing module:
   ```bash
   sudo cp /lib/modules/5.15.136-tegra/kernel/drivers/media/usb/uvc/uvcvideo.ko /lib/modules/5.15.136-tegra/kernel/drivers/media/usb/uvc/uvcvideo.ko.bak
   ```
3. Replace it with the patched version:
   ```bash
   sudo cp path/to/new/uvcvideo.ko /lib/modules/$(uname -r)/kernel/drivers/media/usb/uvc/
   sudo depmod -a
   sudo modprobe uvcvideo
   ```

---

## Troubleshooting Guide: Running Multiple USB Cameras on Linux and Overcoming Bandwidth Limitations

### 1. Problem Statement
Users often encounter errors like `Not enough bandwidth for new device state` or failures to initialize devices when running multiple USB cameras simultaneously on Linux. This issue persists even with compressed formats like MJPEG or low resolutions, preventing scaling up applications requiring multiple video streams.

### 2. Root Cause Analysis: USB Bandwidth Reservation
- **Shared Bandwidth**: USB controllers have finite bandwidth shared among connected devices.
- **Isochronous Transfers**: Video streaming uses USB isochronous transfers, requiring guaranteed bandwidth.
- **Peak Bandwidth Reservation**: USB controllers reserve bandwidth based on the peak rate declared by the camera firmware, even if the actual data rate is lower.
- **Problematic Firmware**: Some cameras declare excessively high peak bandwidth requirements, saturating the USB controller's capacity.

### 3. Standard Troubleshooting & Solutions
- **Check USB Topology**: Use `lsusb -t` to identify how cameras are connected.
- **Separate Devices**: Connect cameras to different USB root hubs or controllers.
- **Use Compression**: Switch to compressed formats like MJPEG or H.264.
- **Lower Resolution/Framerate**: Request lower settings to reduce bandwidth.
- **Use uvcvideo Quirks**: Apply kernel module parameters like `quirks=...`.

### 4. Kernel Patch for uvcvideo (If the above not work or suit for you)
To address the issue of excessive bandwidth reservation, a kernel patch was applied to modify the `uvcvideo` driver:
- **Goal**: Override the `dwMaxPayloadTransferSize` during negotiation for compressed formats.
- **Patch Code**:
  ```c
  // in the last part of function uvc_fixup_video_ctrl()
  if (format->flags & UVC_FMT_FLAG_COMPRESSED) {
      u32 bandwidth = 512 * 8;
      if (stream->dev->udev->speed == USB_SPEED_HIGH)
          bandwidth /= 8;
      ctrl->dwMaxPayloadTransferSize = bandwidth;
  }
  ```
- **Build Process**:
  1. Obtain kernel source matching the running kernel (`uname -r`).
  2. Modify `drivers/media/usb/uvc/uvc_video.c` as shown above.
  3. Update the `Makefile` to ensure `uvcvideo` is built as a module:
     ```makefile
     obj-m += uvcvideo.o
     uvcvideo-objs := uvc_driver.o uvc_queue.o uvc_v4l2.o uvc_video.o uvc_ctrl.o \
                      uvc_status.o uvc_isight.o uvc_debugfs.o uvc_metadata.o
     ifeq ($(CONFIG_MEDIA_CONTROLLER),y)
     uvcvideo-objs += uvc_entity.o
     endif
     obj-$(CONFIG_USB_VIDEO_CLASS) += uvcvideo.o
     all:
         make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
     clean:
         make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
     ```
  4. Build the module:
     ```bash
     make
     ```
  5. Install the module as described in the "Quick Start" section.

### 5. Outcome
The patch bypasses the excessive bandwidth reservation, allowing multiple cameras to stream simultaneously. For example, 8 cameras (4 per USB controller) streaming 1280x720 MJPEG @ 30fps were successfully launched using GStreamer pipelines.

### 6. Warnings and Considerations
- This patch is specific to buggy camera firmware and affects all UVC devices using compressed formats.
- The fixed bandwidth value might be too low for other cameras or higher-quality streams.
- Kernel modifications require rebuilding/reinstalling the module after updates.
- This workaround bypasses standard USB bandwidth management and may cause instability if actual data rates exceed the forced low payload size.
