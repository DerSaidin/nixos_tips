### Firefox use video decode NVIDIA

https://www.askwoody.com/forums/topic/firefox-video-decode-acceleration-how-did-i-miss-this/

`about:config`

Change:
```
media.ffmpeg.dmabuf-textures.enabled = true
media.ffmpeg.vaapi.enabled = true
media.ffvpx.enabled = false
```

Set environment variable:
```
// X11:
MOZ_X11_EGL=1

// For Wayland:
MOZ_ENABLE_WAYLAND=1
```

