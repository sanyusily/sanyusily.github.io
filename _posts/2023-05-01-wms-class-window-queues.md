---
layout: article
title: 'WindowManagerService WMS'
tags: code wms
---

A video with Header Image, See [![Watch the video](http://i0.hdslb.com/bfs/archive/db72e31115aa27779ffd0043ab76facd6ae5c2de.jpg)](https://www.bilibili.com/video/BV1Av411J7ad)  for more examp  es.



修复第三方一些垃圾app 透明窗口盖在app上面使功能失效

```terminal
diff --git a/native/services/inputflinger/InputDispatcher.cpp b/native/services/inputflinger/InputDispatcher.cpp
index 10290cf9..52c1a68c 100755
--- a/native/services/inputflinger/InputDispatcher.cpp
+++ b/native/services/inputflinger/InputDispatcher.cpp
@@ -522,6 +522,11 @@ sp<InputWindowHandle> InputDispatcher::findTouchedWindowAtLocked(int32_t display
                     bool isTouchModal = (flags & (InputWindowInfo::FLAG_NOT_FOCUSABLE
                             | InputWindowInfo::FLAG_NOT_TOUCH_MODAL)) == 0;
                     if (isTouchModal || windowInfo->touchableRegionContainsPoint(x, y)) {
+                        ALOGD(" InputDispatcher findTouchedWindowAtLocked %s finded  ",windowHandle->getName().c_str());
+                        if (windowHandle->getName().find("142 dpi") != std::string::npos) {
+                            ALOGD("%s do not accept any motion ",windowHandle->getName().c_str());
+                            continue;
+                        }
                         int32_t portalToDisplayId = windowInfo->portalToDisplayId;
                         if (portalToDisplayId != ADISPLAY_ID_NONE
                                 && portalToDisplayId != displayId) {
```





