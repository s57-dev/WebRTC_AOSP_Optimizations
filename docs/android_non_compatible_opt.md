## WebView Single process mode



By default, the WebView process on multi-arch devices will start up with 32bit primary ABI. This could lead some performance degradation on 64bit ARM devices. In addition, with multiple encoders/decoders the IPC mechanism between the app and the WebView rendering process will introduce some noticeable CPU load. or closed, non-Android compliant devices, enabling single process mode could be beneficial. However, this type of change needs to be applied with caution, since it circumvents essential WebView security mechanism. It should be never enabled on devices that potentially open untrusted URLs.

To disable/enable multi-process mode:

```bash
(adb) $ cmd webviewupdate disable-multiprocess 
(adb) $ cmd webviewupdate enable-multiprocess 
```

After disabling multi-process mode the WebView libraries will be always loaded with an ABI matching the application using WebView.

