---
title: 用Avfoundation 录像
date: 2015-10-27 23:40:23
tags:
- avfoundation 
- iOS
---

参考了[Capturing Video on iOS](https://www.objc.io/issues/23-video/capturing-video/), [CUSTOM CAMERA ON IOS – AVCAPTURESESSION, AVCAPTUREVIDEOPREVIEWLAYER TUTORIAL](https://dannygtech.wordpress.com/2014/03/04/custom-camera-on-ios-avcapturesession-avcapturevideopreviewlayer-tutorial/)。（吐槽一下[Capturing Video on iOS](https://www.objc.io/issues/23-video/capturing-video/)… sample code 搞那么复杂干嘛。。）

#### 先开一个新的AVCaptureSession
```objc
AVCaptureSession *captureSession = [AVCaptureSession new];
[captureSession setSessionPreset:AVCaptureSessionPresetHigh];// the video quiality
```
#### 然后加入audio 和video inputs
audio
```objc
AVAudioSession *audioSession = [AVAudioSession sharedInstance];
[audioSession setCategory:AVAudioSessionCategoryPlayAndRecord error:nil];//allow recording and replaying at the same time
[audioSession setActive:YES error:nil];
AVCaptureDeviceInput *micDeviceInput = [[AVCaptureDeviceInput alloc] initWithDevice:[AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio] error:&error];
if ([captureSession canAddInput:micDeviceInput]) {
    [captureSession addInput:micDeviceInput];
}
```
video
```
AVCaptureDevice *cameraDevice = nil;
NSArray *devices = [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo];

// make it the front camera
for (AVCaptureDevice *device in devices) {
    if ([device position] == AVCaptureDevicePositionFront) {
        cameraDevice = device;
    }
}

AVCaptureDeviceInput *cameraDeviceInput = [[AVCaptureDeviceInput alloc] initWithDevice:cameraDevice error:&error];
if ([captureSession canAddInput:cameraDeviceInput]) {
    [captureSession addInput:cameraDeviceInput];
}
```
#### 然后把output也连上
```objc
AVCaptureMovieFileOutput *movieFileOutput = [AVCaptureMovieFileOutput new];
if([captureSession canAddOutput:movieFileOutput]){
    [captureSession addOutput:movieFileOutput];
}
```
#### 最后再加个显示live preview 的layer
```objc
AVCaptureVideoPreviewLayer *previewLayer = [[AVCaptureVideoPreviewLayer alloc] initWithSession:captureSession];
UIView *camView = [[UIView alloc] initWithFrame:CGRectMake(...)];
previewLayer.frame = camView.bounds;
previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
[camView.layer addSublayer:previewLayer];
[self.view addSubview:camView];
```
#### 开始录像
```objc
NSURL *outputURL = [NSURL fileURLWithPath:[NSString stringWithFormat:@"%@tmp.mov", NSTemporaryDirectory()]];
[captureSession startRunning];
[movieFileOutput startRecordingToOutputFileURL:outputURL recordingDelegate:self];
```
#### 结束&保存
```objc
[_captureSession stopRunning];
[_movieFileOutput stopRecording];
UISaveVideoAtPathToSavedPhotosAlbum(_movieFileOutput.outputFileURL.path, nil, NULL, NULL);
```
完🐶


