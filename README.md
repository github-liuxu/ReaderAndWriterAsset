# ReaderAndWriterAsset
AVFoundation AVAssetReader和AVAssetWriter
    AVAsset *asset = [AVAsset assetWithURL:[NSURL fileURLWithPath:path]];
    NSError *error;
    //创建AVAssetReader对象用来读取asset数据
    assetReader = [AVAssetReader assetReaderWithAsset:asset error:&error];
    AVAsset *localAsset = assetReader.asset;

    AVAssetTrack *videoTrack = [[localAsset tracksWithMediaType:AVMediaTypeVideo] firstObject];
    AVAssetTrack *audioTrack = [[localAsset tracksWithMediaType:AVMediaTypeAudio] firstObject];

    NSDictionary *videoSetting = @{(id)kCVPixelBufferPixelFormatTypeKey     : [NSNumber numberWithUnsignedInt:kCVPixelFormatType_32BGRA],
                                   (id)kCVPixelBufferIOSurfacePropertiesKey : [NSDictionary dictionary],
                                   };
//    AVAssetReaderTrackOutput用来设置怎么读数据
    readerVideoTrackOutput = [[AVAssetReaderTrackOutput alloc] initWithTrack:videoTrack outputSettings:videoSetting];
    //音频以pcm流的形似读数据
    NSDictionary *audioSetting = @{AVFormatIDKey : [NSNumber numberWithUnsignedInt:kAudioFormatLinearPCM]};
    readerAudioTrackOutput = [[AVAssetReaderTrackOutput alloc] initWithTrack:audioTrack outputSettings:audioSetting];
    
    if ([assetReader canAddOutput:readerVideoTrackOutput]) {
        [assetReader addOutput:readerVideoTrackOutput];
    }
    
    if ([assetReader canAddOutput:readerAudioTrackOutput]) {
        [assetReader addOutput:readerAudioTrackOutput];
    }
    //开始读
    [assetReader startReading];
    
    NSURL *writerUrl = [NSURL fileURLWithPath:writerPath];
    //创建一个写数据对象
    assetWriter = [[AVAssetWriter alloc] initWithURL:writerUrl fileType:AVFileTypeMPEG4 error:nil];
    //配置写数据，设置比特率，帧率等
    NSDictionary *compressionProperties = @{ AVVideoAverageBitRateKey : @(1.38*1024*1024),
                                             AVVideoExpectedSourceFrameRateKey: @(30),
                                             AVVideoProfileLevelKey : AVVideoProfileLevelH264HighAutoLevel };
    //配置编码器宽高等
    NSDictionary *compressionVideoSetting = @{
                              AVVideoCodecKey                   : AVVideoCodecTypeH264,
                              AVVideoWidthKey                   : @1080,
                              AVVideoHeightKey                  : @1080,
                              AVVideoCompressionPropertiesKey   : compressionProperties
                              };

    AudioChannelLayout stereoChannelLayout = {
        .mChannelLayoutTag = kAudioChannelLayoutTag_Stereo,
        .mChannelBitmap = 0,
        .mNumberChannelDescriptions = 0
    };
    NSData *channelLayoutAsData = [NSData dataWithBytes:&stereoChannelLayout length:offsetof(AudioChannelLayout, mChannelDescriptions)];
    //写入音频配置
    NSDictionary *compressionAudioSetting = @{
                                               AVFormatIDKey         : [NSNumber numberWithUnsignedInt:kAudioFormatMPEG4AAC],
                                               AVEncoderBitRateKey   : [NSNumber numberWithInteger:64000],
                                               AVSampleRateKey       : [NSNumber numberWithInteger:44100],
                                               AVChannelLayoutKey    : channelLayoutAsData,
                                               AVNumberOfChannelsKey : [NSNumber numberWithUnsignedInteger:2]
                                               };
    //AVAssetWriterInput用来说明怎么写数据
    assetVideoWriterInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeVideo outputSettings:compressionVideoSetting];
    assetAudioWriterInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeAudio outputSettings:compressionAudioSetting];
    
    if ([assetWriter canAddInput:assetVideoWriterInput]) {
        [assetWriter addInput:assetVideoWriterInput];
    }
    if ([assetWriter canAddInput:assetAudioWriterInput]) {
        [assetWriter addInput:assetAudioWriterInput];
    }
    //开始写
    [assetWriter startWriting];
    [assetWriter startSessionAtSourceTime:kCMTimeZero];
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t videoWriter = dispatch_queue_create("videoWriter", DISPATCH_QUEUE_CONCURRENT);
    dispatch_queue_t audioWriter = dispatch_queue_create("audioWriter", DISPATCH_QUEUE_CONCURRENT);
    __block BOOL isVideoComplete = NO;
    dispatch_group_enter(group);
    //要想写数据，就带有数据源，以下是将readerVideoTrackOutput读出来的数据加入到assetVideoWriterInput中再写入本地，音频和视频读取写入方式一样
    [assetVideoWriterInput requestMediaDataWhenReadyOnQueue:videoWriter usingBlock:^{
        while (!isVideoComplete && assetVideoWriterInput.isReadyForMoreMediaData) {
            //样本数据
            @autoreleasepool {
                //每次读取一个buffer
                CMSampleBufferRef buffer = [readerVideoTrackOutput copyNextSampleBuffer];
                if (buffer) {
                    //将读来的buffer加入到写对象中开始写
                    //此处也可以给assetVideoWriterInput加个适配器对象可以写入CVPixelBuffer
                    [assetVideoWriterInput appendSampleBuffer:buffer];
                    //将buffer生成图片
                    UIImage *image = [self imageFromSampleBuffer:buffer];
                    //给图片添加滤镜
                    UIImage *img = [self imageAddFilter:image];
                    dispatch_async(dispatch_get_main_queue(), ^{
                        self.imageView.image = img;
                    });
                    CFRelease(buffer);
                    buffer = NULL;
                } else {
                    isVideoComplete = YES;
                }
            }
            
        }
        if (isVideoComplete) {
            //关闭写入会话
            [assetVideoWriterInput markAsFinished];
            dispatch_group_leave(group);
        }
    }];
