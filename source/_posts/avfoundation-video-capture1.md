---
title: 用Avfoundation 录像(续)
date: 2015-12-06 12:29:05
tags:
- avfoundation
- iOS
---
除了直接用AVCaptureMovieFileOutput，还可以把AVCaptureVideoDataOutput，AVCaptureAudioDataOutput 当作output。这两个class的delegate AVCaptureVideoDataOutputSampleBufferDelegate, AVCaptureAudioDataOutputSampleBufferDelegate 有一个- (void)captureOutput:(AVCaptureOutput )captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection )connection 的method, 开始录像之后你可以从这个method 里面拿到每一个录下的frame，然后可以用AVAssetWriter来写入local的文件。

1. 加output
```objc
_videoDataOutput = [AVCaptureVideoDataOutput new];
_audioDataOutput = [AVCaptureAudioDataOutput new];
if ([_captureSession canAddOutput:_videoDataOutput]) {
    DLog(@"added video out");
    [_captureSession addOutput:_videoDataOutput];
}
_videoConnection = [_videoDataOutput connectionWithMediaType:AVMediaTypeVideo];
_videoConnection.videoOrientation = AVCaptureVideoOrientationPortrait;

[_videoDataOutput setSampleBufferDelegate:self queue:_videoDataOutputQueue];

if ([_captureSession canAddOutput:_audioDataOutput]) {
    DLog(@"added audio out");
    [_captureSession addOutput:_audioDataOutput];
}
_audioConnection = [_audioDataOutput connectionWithMediaType:AVMediaTypeAudio];
[_audioDataOutput setSampleBufferDelegate:self queue:_audioDataOutputQueue];
```

2. 准备AVAssetWriter, 需要两个writerInput 分别写入video 和audio
```objc
_assetWriter = [AVAssetWriter assetWriterWithURL:[self outputURL] fileType:AVFileTypeMPEG4 error:nil];
_videoAssetWriterInput = [[AVAssetWriterInput alloc] initWithMediaType:AVMediaTypeVideo
                                                                outputSettings:[_videoDataOutput recommendedVideoSettingsForAssetWriterWithOutputFileType:AVFileTypeMPEG4]];
_videoAssetWriterInput.expectsMediaDataInRealTime = YES;
_audioAssetWriterInput = [[AVAssetWriterInput alloc] initWithMediaType:AVMediaTypeAudio
                                                                outputSettings:[_audioDataOutput recommendedAudioSettingsForAssetWriterWithOutputFileType:AVFileTypeMPEG4]];
_audioAssetWriterInput.expectsMediaDataInRealTime = YES;

if ([_assetWriter canAddInput:_videoAssetWriterInput]) {
    DLog(@"added video out writer");
    [_assetWriter addInput:_videoAssetWriterInput];
}
if ([_assetWriter canAddInput:_audioAssetWriterInput]) {
    DLog(@"added audio out writer");
    [_assetWriter addInput:_audioAssetWriterInput];
}
```
3. 填写 - (void)captureOutput:(AVCaptureOutput )captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection )connection
```objc
- (void)captureOutput:(AVCaptureOutput *)captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection
{
    CFRetain(sampleBuffer);
    CMFormatDescriptionRef formatDesc = CMSampleBufferGetFormatDescription(sampleBuffer);
    CMMediaType mediaType = CMFormatDescriptionGetMediaType(formatDesc);
    CMTime currentSampleTime = CMSampleBufferGetPresentationTimeStamp(sampleBuffer);

    if (mediaType == kCMMediaType_Video) {
        if(_firstSample) {
            if ([_assetWriter startWriting]) {
                [_assetWriter startSessionAtSourceTime:currentSampleTime];
                _firstSample = NO;
            } else {
                DLog(@"Error when starting avseet writer, %ld %@, %@", (long)_assetWriter.status, [_assetWriter.error localizedDescription], [_assetWriter.error userInfo]);
            }
            return;
        }

        if (_videoAssetWriterInput.readyForMoreMediaData) {
            [_videoAssetWriterInput appendSampleBuffer:bufferToWrite];
        }
    } else if (mediaType == kCMMediaType_Audio && !_firstSample) {
        if (_audioAssetWriterInput.readyForMoreMediaData) {
            [_audioAssetWriterInput appendSampleBuffer:bufferToWrite];
        }
    }
    
    CFRelease(sampleBuffer);
}
```

4. 在didOutputSampleBuffer里面拿到sample buffer 之后还可以对其进行各种处理－－将其转化为CIImage ，然后就可以apply 各种filter。
```objc
CFRetain(pixelBuffer);
CVPixelBufferLockBaseAddress(pixelBuffer, 0 );

CIImage *img = [CIImage imageWithCVPixelBuffer:pixelBuffer];
CIFilter *filter = [CIFilter filterWithName:<filterName>]; //具体 filter可以参考[Core Image Filter Reference](https://developer.apple.com/library/prerelease/mac/documentation/GraphicsImaging/Reference/CoreImageFilterReference/index.html)

... apply filters

CIImage *outputImage = [filter outputImage];
[_ciContext render:outputImage toCVPixelBuffer:pixelBuffer]; //把处理好的ciimage 再写回buffer

CVPixelBufferUnlockBaseAddress(pixelBuffer, 0);
CFRelease(pixelBuffer);
```

5. 那么live preview 呢，其实如果没有别的要求的话可以直接用之前用的AVCaptureVideoPreviewLayer，但是如果要在live preview 里面显示filter 的效果的话。那么就要将frame 画到一个GLKView 上面，然后把这个view 展示给user.

   create GLKView。这里要用 EAGLContext来create CIContext 这样具体draw的时候就会用GPU 来draw 了，否则将使用CPU 来画
   ```objc
   _eaglContext = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
   _ciContext = [CIContext contextWithEAGLContext:_eaglContext options:@{kCIContextWorkingColorSpace : [NSNull null]} ];
       
   GLKView *glkView = [[GLKView alloc] initWithFrame:<frame> context:_eaglContext];
   [glkView bindDrawable];
   _livePreviewView = glkView;
   ```
   画frame
   ```objc
   CFRetain(pixelBuffer);
   CVPixelBufferLockBaseAddress(pixelBuffer, 0 );
   
   CIImage *img = [CIImage imageWithCVPixelBuffer:pixelBuffer];
   
   [_ciContext drawImage:outputImage inRect:previewRect fromRect:cropRect];
   [((GLKView *)_livePreviewView) display];
   
   CVPixelBufferUnlockBaseAddress(pixelBuffer, 0);
   CFRelease(pixelBuffer);
   ```

6. 最后保存到Library
```objc
[_captureSession stopRunning];
[_videoAssetWriterInput markAsFinished];
[_audioAssetWriterInput markAsFinished];

[_assetWriter finishWritingWithCompletionHandler:^{
    ALAssetsLibrary *library = [[ALAssetsLibrary alloc] init];
    if ([library videoAtPathIsCompatibleWithSavedPhotosAlbum:videoURL]) {
        [library writeVideoAtPathToSavedPhotosAlbum:videoURL completionBlock:^(NSURL *assetURL, NSError *error){
            if (error) {
                ALog(@"Error when saving captured video to library, error = %@, url = %@", error, assetURL);
            }
        }];
    }
}];
```


