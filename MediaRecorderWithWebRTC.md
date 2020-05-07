# About

How to use this forked MediaRecorder plugin with iosrtc plugin to record audio stream/track to audio stream (chunk data).
Support audio only, not support video yet.

## Installation

media-recorder and iosrtc

```bash
cordova plugin add https://github.com/pinionpi/phonegap-plugin-media-recorder.git
cordova plugin add cordova-plugin-iosrtc

cordova platform rm ios
cordova platform add ios
```

## Conflict plugins

- Not use media-stream because intend to use iosrtc as a replacement.
- Not use cordova-plugin-media because of duplicate symbol '_OBJC_CLASS_$_CDVAudioRecorder' compile error.

```bash
cordova plugin rm phonegap-plugin-media-stream
cordova plugin rm cordova-plugin-media
```

## Sample code

Record audio mediaStream for 5 seconds, fire ondataavailable every time slice 100ms.

```javascript

// Use gum of iosrtc / navigator.mediaDevices.getUserMedia
cordova.plugins.iosrtc.registerGlobals();

// 'audio/m4a' or 'audio/wav'
var mimeType = 'audio/wav'; // WAV 29372 bytes per 100ms / 878 KB for 10s
//var mimeType = 'audio/m4a'; // AAC 11213 bytes per 100ms / 135 KB for 10s (compressed/small)
var timeSlice = 100;

navigator.mediaDevices.getUserMedia({
    'audio': true,
    'video': false
}).then(function(mediastream) {
    var options = { mimeType : mimeType};
    window.mediaRecorder = new MediaRecorder(mediastream, options);

    // ondataavailable slice
    mediaRecorder.ondataavailable = function(blob) {
        //console.log('ondataavailable: ' + blob.size, blob);
        window.blob = blob;
        window.chunkedBlob = blob.slice(window.chunkedLast || 0);
        window.chunkedLast = blob.size;
        if (!window.chunkedBlob.size) {
            return;
        }
        // Blob2ArrayBuffer
        fileReader = new FileReader();
        fileReader.onload = function() {
            var buf = this.result;
            var u8 = new Uint8Array(buf);
            console.log("chunk " + buf.byteLength + " bytes. e.g. first byte is " + u8[0]); // , u8
        };
        fileReader.readAsArrayBuffer(chunkedBlob);
    };

    // poc recorder timeSlice for x seconds, then stop
    // Use start + setInterval requestData
    mediaRecorder.start();

    // NOTE: You may need to modify mediaRecorder.src to work with Ionic's WKWebView => try ionicNormalizer
    // fetch => PASSED
    // resolveLocalFileSystemURL => FAILED
    // mediaRecorder.src = ionicNormalizer(mediaRecorder.src);

    window.mediaRecorderTimer = setInterval(function() {
        mediaRecorder.requestData();
    }, timeSlice);

    setTimeout(function() {
        clearInterval(window.mediaRecorderTimer);
        mediaRecorder.stop();
        console.log("stop recording");
    }, 5000);

});

```

### TODO cdvfile

1. Work with WkWebView

Just workaround in the sample code to use ionicNormalizer to successfully fetch for Blob.

```javascript
var ionicNormalizer = (window.Ionic.WebView && window.Ionic.WebView.convertFileSrc) || window.Ionic.normalizeURL;
mediaRecorder.src = ionicNormalizer(mediaRecorder.src);
```

etc.

```
CSP connect-src cdvfile:
fetch mediaRecorder.src
window.resolveLocalFileSystemURL(cordova.file.tempDirectory, ..);
```

2. Multiple recording sessions

Not support yet. TODO separate append and fetch temporary files

Current file path can be:

- cdvfile://localhost/temporary/recording.m4a
- cdvfile://localhost/temporary/recording.wav

### Output processing

Main goal of recorder into chunked is for playback in realtime.

- You can transfer chunked data via desired medium, and playback at destination.
- Optional steps may be encode/decode data into other format.
- For playback wave, just try wrapWaveHeader, decodeAudioData to get source buffer, and connect audio source node to destnation.

### AAC patent infringement

If you use audio/m4a mimeType, please read wiki about [AAC codecs](https://en.wikipedia.org/wiki/Advanced_Audio_Coding).

> However, a patent license is[when?] required for all manufacturers or developers of AAC codecs.[49] For this reason, free and open source software implementations such as FFmpeg and FAAC may be distributed in source form only, in order to avoid patent infringement. (See below under Products that support AAC, Software.)
