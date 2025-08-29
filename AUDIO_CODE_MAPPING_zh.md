# 音频数据流代码映射分析

本文档提供音频数据流中每个步骤在代码库中的具体位置和实现细节。

## 1. 应用层接口 (Application Layer)

### 1.1 AudioRenderer创建和写入
**文件**: `interfaces/inner_api/native/audiorenderer/include/audio_renderer.h`
```cpp
class AudioRenderer {
    // 写入音频数据的主要接口
    virtual int32_t Write(uint8_t *buffer, size_t bufferSize, bool isBlocking = true) = 0;
    
    // 获取缓冲区描述符（回调模式）
    virtual int32_t GetBufferDesc(BufferDesc &bufDesc) const = 0;
    
    // 入队音频数据（回调模式）  
    virtual int32_t Enqueue(const BufferDesc &bufDesc) const = 0;
}
```

**实现位置**: `frameworks/native/audiorenderer/src/audio_renderer.cpp`
```cpp
// AudioRendererPrivate::Write() - 第1348行
int32_t AudioRendererPrivate::Write(uint8_t *buffer, size_t bufferSize, bool isBlocking)
{
    Trace trace("AudioRenderer::Write");
    std::shared_ptr<IAudioStream> currentStream = GetInnerStream();
    CHECK_AND_RETURN_RET_LOG(currentStream != nullptr, ERROR_ILLEGAL_STATE, "audioStream_ is nullptr");
    return currentStream->Write(buffer, bufferSize, isBlocking);
}

// AudioRendererPrivate::Enqueue() - 第1728行  
int32_t AudioRendererPrivate::Enqueue(const BufferDesc &bufDesc)
{
    Trace trace("AudioRenderer::Enqueue");
    AsyncCheckAudioRenderer("Enqueue");
    MockPcmData(bufDesc.buffer, bufDesc.bufLength);
    DumpFileUtil::WriteDumpFile(dumpFile_, static_cast<void *>(bufDesc.buffer), bufDesc.bufLength);
    std::shared_ptr<IAudioStream> currentStream = audioStream_;
    CHECK_AND_RETURN_RET_LOG(currentStream != nullptr, ERROR_ILLEGAL_STATE, "audioStream_ is nullptr");
    int32_t ret = currentStream->Enqueue(bufDesc);
    return ret;
}
```

## 2. 框架层流管理 (Framework Stream Management)

### 2.1 音频流接口
**文件**: `frameworks/native/audiostream/include/i_audio_stream.h`
```cpp
class IAudioStream {
public:
    virtual int32_t Write(uint8_t *buffer, size_t bufferSize, bool isBlocking = true) = 0;
    virtual int32_t Enqueue(const BufferDesc &bufDesc) = 0;
    virtual StreamClass GetStreamClass() = 0;
}
```

### 2.2 流类型映射
**文件**: `frameworks/native/audiorenderer/src/audio_renderer.cpp` (第84-97行)
```cpp
static const std::map<uint32_t, IAudioStream::StreamClass> AUDIO_OUTPUT_FLAG_GROUP_MAP = {
    {AUDIO_OUTPUT_FLAG_NORMAL, IAudioStream::StreamClass::PA_STREAM},      // 普通流
    {AUDIO_OUTPUT_FLAG_DIRECT, IAudioStream::StreamClass::PA_STREAM},      // 直通流
    {AUDIO_OUTPUT_FLAG_MULTICHANNEL, IAudioStream::StreamClass::PA_STREAM}, // 多声道流
    {AUDIO_OUTPUT_FLAG_LOWPOWER, IAudioStream::StreamClass::PA_STREAM},    // 低功耗流
    {AUDIO_OUTPUT_FLAG_FAST, IAudioStream::StreamClass::FAST_STREAM},      // 快速流
    {AUDIO_OUTPUT_FLAG_HWDECODING, IAudioStream::StreamClass::PA_STREAM},  // 硬件解码流
};
```

## 3. 服务层流实现 (Service Layer Stream Implementation)

### 3.1 ProRendererStreamImpl - 主要渲染流实现
**文件**: `services/audio_service/server/src/pro_renderer_stream_impl.cpp`

```cpp
// 构造函数 - 第42行
ProRendererStreamImpl::ProRendererStreamImpl(AudioProcessConfig processConfig, bool isDirect)
    : isDirect_(isDirect), isNeedResample_(false), isNeedMcr_(false)
{
    // 初始化流参数
}

// 写入接口实现
int32_t ProRendererStreamImpl::Write(uint8_t *buffer, size_t bufferSize, bool isBlocking)
{
    // 执行数据写入和处理
    return ProcessAndWrite(buffer, bufferSize);
}

// 采样率直通处理 - 第76行
AudioSamplingRate ProRendererStreamImpl::GetDirectSampleRate(AudioSamplingRate sampleRate) const noexcept
{
    if (processConfig_.streamType == STREAM_VOICE_CALL || processConfig_.streamType == STREAM_VOICE_COMMUNICATION) {
        // VoIP流类型特殊处理
        if (sampleRate <= AudioSamplingRate::SAMPLE_RATE_16000) {
            return AudioSamplingRate::SAMPLE_RATE_16000;
        } else {
            return AudioSamplingRate::SAMPLE_RATE_48000;
        }
    }
    // 音乐流高分辨率处理
    switch (sampleRate) {
        case AudioSamplingRate::SAMPLE_RATE_44100:
            return AudioSamplingRate::SAMPLE_RATE_48000;
        case AudioSamplingRate::SAMPLE_RATE_88200:
            return AudioSamplingRate::SAMPLE_RATE_96000;
        // ... 更多转换规则
    }
}
```

### 3.2 音频数据缓冲管理
**文件**: `services/audio_service/server/src/pro_renderer_stream_impl.cpp` (第521-634行)
```cpp
// 获取缓冲区状态
int32_t ProRendererStreamImpl::GetBufQueueState(BufferQueueState &bufState) const;

// 从缓冲区读取数据
int32_t ProRendererStreamImpl::Peek(std::vector<char> *audioBuffer, int32_t &index);

// 弹出写缓冲区索引
int32_t ProRendererStreamImpl::PopWriteBufferIndex();

// 弹出sink缓冲区数据
void ProRendererStreamImpl::PopSinkBuffer(std::vector<char> *audioBuffer, int32_t &index);
```

## 4. 音频策略服务 (Audio Policy Service)

### 4.1 设备管理和采样率获取
**文件**: `services/audio_policy/server/service/service_main/src/audio_policy_service.cpp` (第791-812行)
```cpp
int32_t AudioPolicyService::GetHardwareOutputSamplingRate(const std::shared_ptr<AudioDeviceDescriptor> &desc)
{
    int32_t rate = 48000; // 默认采样率
    
    CHECK_AND_RETURN_RET_LOG(desc != nullptr, -1, "desc is null!");
    
    bool ret = audioConnectedDevice_.IsConnectedOutputDevice(desc);
    CHECK_AND_RETURN_RET(ret, -1);
    
    // 根据设备类型获取支持的采样率
    std::unordered_map<ClassType, std::list<AudioModuleInfo>> deviceClassInfo = {};
    audioConfigManager_.GetDeviceClassInfo(deviceClassInfo);
    DeviceType clientDevType = desc->deviceType_;
    
    for (const auto &device : deviceClassInfo) {
        auto moduleInfoList = device.second;
        for (auto &moduleInfo : moduleInfoList) {
            auto serverDevType = AudioPolicyUtils::GetInstance().GetDeviceType(moduleInfo.name);
            if (clientDevType == serverDevType) {
                rate = atoi(moduleInfo.rate.c_str());
                return rate;
            }
        }
    }
    return rate;
}
```

### 4.2 管道管理
**文件**: `services/audio_policy/server/service/service_main/src/audio_core_service_private.cpp` (第692-2751行)
```cpp
// 输出管道创建
void AudioCoreService::ProcessOutputPipeNew(std::shared_ptr<AudioPipeInfo> pipeInfo, uint32_t &flag,
    const AudioStreamDeviceChangeReasonExt reason);

// 输入管道创建  
void AudioCoreService::ProcessInputPipeNew(std::shared_ptr<AudioPipeInfo> pipeInfo, uint32_t &flag);

// 助听器模块加载
int32_t AudioCoreService::LoadHearingAidModule(DeviceType deviceType, const AudioStreamInfo &audioStreamInfo,
    std::string networkId, std::string sinkName, SourceType sourceType)
{
    // 创建管道信息
    std::shared_ptr<AudioPipeInfo> pipeInfo = std::make_shared<AudioPipeInfo>();
    pipeInfo->id_ = ioHandle;
    pipeInfo->paIndex_ = paIndex;
    pipeInfo->name_ = "hearing_aid_output";
    pipeInfo->pipeRole_ = PIPE_ROLE_OUTPUT;
    pipeInfo->routeFlag_ = AUDIO_OUTPUT_FLAG_NORMAL;
    pipeInfo->adapterName_ = "hearing_aid";
    pipeInfo->moduleInfo_ = moduleInfo;
    pipeInfo->pipeAction_ = PIPE_ACTION_DEFAULT;
    pipeManager_->AddAudioPipeInfo(pipeInfo);
}
```

## 5. 音频引擎处理 (Audio Engine Processing)

### 5.1 格式转换节点
**文件**: `services/audio_engine/node/src/hpae_audio_format_converter_node.cpp` (第112-163行)
```cpp
int32_t HpaeAudioFormatConverterNode::ConverterProcess(float *srcData, float *dstData, float *tmpData,
    HpaePcmBuffer *input)
{
    // 检查是否需要转换
    if ((inChannelInfo.numChannels == outChannelInfo.numChannels) && (inRate == outRate)) {
        // 无需转换，直接拷贝
        ret = memcpy_s(dstData, outputFrameBytes, srcData, inputFrameBytes);
    } else if (inChannelInfo.numChannels == outChannelInfo.numChannels) {
        // 仅重采样
        ret = resampler_->Process(srcData, inputFrameLen, dstData, outputFrameLen);
    } else if (inRate == outRate) {
        // 仅声道转换
        ret = channelConverter_.Process(inputFrameLen, srcData, (*input).Size(), dstData, converterOutput_.Size());
    } else if (inChannelInfo.numChannels > outChannelInfo.numChannels) {
        // 先声道转换，后重采样
        ret = channelConverter_.Process(inputFrameLen, srcData, (*input).Size(), tmpData, tmpOutBuf_.Size());
        ret += resampler_->Process(tmpData, inputFrameLen, dstData, outputFrameLen);
    } else {
        // 先重采样，后声道转换
        ret = resampler_->Process(srcData, inputFrameLen, tmpData, outputFrameLen);
        ret += channelConverter_.Process(outputFrameLen, tmpData, tmpOutBuf_.Size(), dstData, converterOutput_.Size());
    }
    return ret;
}

HpaePcmBuffer *HpaeAudioFormatConverterNode::SignalProcess(const std::vector<HpaePcmBuffer *> &inputs)
{
    float *srcData = inputs[0]->GetPcmDataBuffer();
    float *dstData = converterOutput_.GetPcmDataBuffer();
    float *tmpData = tmpOutBuf_.GetPcmDataBuffer();
    
    if (resampler_ == nullptr) {
        return &silenceData_;
    }
    
    int32_t ret = ConverterProcess(srcData, dstData, tmpData, inputs[0]);
    if (ret != EOK) {
        AUDIO_ERR_LOG("NodeId %{public}d, sessionId %{public}d, Format Converter fail to process!",
            GetNodeId(), GetSessionId());
        return &silenceData_;
    }
    
    return &converterOutput_;
}
```

### 5.2 音频输出节点  
**文件**: `services/audio_engine/node/src/hpae_sink_output_node.cpp` (第1-100行)
```cpp
// 构造函数
HpaeSinkOutputNode::HpaeSinkOutputNode(HpaeNodeInfo &nodeInfo)
    : HpaeNode(nodeInfo),
      renderFrameData_(nodeInfo.frameLen * nodeInfo.channels * GetSizeFromFormat(nodeInfo.format)),
      interleveData_(nodeInfo.frameLen * nodeInfo.channels)
{
    AUDIO_INFO_LOG("HpaeSinkOutputNode name is %{public}s", sinkOutAttr_.adapterName.c_str());
}

// 主要处理函数
void HpaeSinkOutputNode::DoProcess()
{
    Trace trace("HpaeSinkOutputNode::DoProcess " + GetTraceInfo());
    if (audioRendererSink_ == nullptr) {
        AUDIO_WARNING_LOG("audioRendererSink_ is nullptr sessionId: %{public}u", GetSessionId());
        return;
    }
    
    // 从输入流读取数据
    std::vector<HpaePcmBuffer *> &outputVec = inputStream_.ReadPreOutputData();
    CHECK_AND_RETURN(!outputVec.empty());
    HpaePcmBuffer *outputData = outputVec.front();
    
    // 功率处理
    HandlePaPower(outputData);
    
    // 格式转换（float到目标格式）
    ConvertFromFloat(GetBitWidth(), GetChannelCount() * GetFrameLen(), 
                     outputData->GetPcmDataBuffer(), renderFrameData_.data());
    
    uint64_t writeLen = 0;
    char *renderFrameData = (char *)renderFrameData_.data();
    
    // 触觉参数处理
    HandleHapticParam(renderFrameTimes_);
    renderFrameTimes_ += MS_PER_FRAME;
    
    // 渲染到硬件
    auto ret = audioRendererSink_->RenderFrame(*renderFrameData, renderFrameData_.size(), writeLen);
    if (ret != SUCCESS || writeLen != renderFrameData_.size()) {
        // 处理错误情况
    }
}
```

## 6. 音频格式转换工具 (Audio Utility Functions)

### 6.1 格式转换函数
**文件**: `frameworks/native/audioutils/src/audio_utils.cpp` (第1302-1464行)
```cpp
// 16位整数到32位整数转换
static void MemcpyToI32FromI16(int16_t *src, int32_t *dst, size_t count)
{
    for (size_t i = 0; i < count; i++) {
        dst[i] = static_cast<int32_t>(src[i]) << 16;
    }
}

// 24位到32位整数转换
static void MemcpyToI32FromI24(uint8_t *src, int32_t *dst, size_t count)
{
    for (size_t i = 0; i < count; i++) {
        int32_t sample = (src[i * 3] << 8) | (src[i * 3 + 1] << 16) | (src[i * 3 + 2] << 24);
        dst[i] = sample;
    }
}

// 32位浮点到32位整数转换
static void MemcpyToI32FromF32(float *src, int32_t *dst, size_t count)
{
    for (size_t i = 0; i < count; i++) {
        dst[i] = static_cast<int32_t>(src[i] * INT32_MAX);
    }
}

// 信号检测代理
bool SignalDetectAgent::CheckAudioData(uint8_t *buffer, size_t bufferLen)
{
    int32_t *cache = cacheAudioData_.data();
    
    // 根据采样格式转换数据
    if (sampleFormat_ == SAMPLE_F32LE) {
        float *cp = reinterpret_cast<float*>(buffer);
        MemcpyToI32FromF32(cp, cache, frameCountIgnoreChannel_);
    } else if (sampleFormat_ == SAMPLE_S24LE) {
        MemcpyToI32FromI24(buffer, cache, frameCountIgnoreChannel_);
    } else {
        int16_t *cp = reinterpret_cast<int16_t*>(buffer);
        MemcpyToI32FromI16(cp, cache, frameCountIgnoreChannel_);
    }
    
    // 检测信号数据
    if (DetectSignalData(cache, frameCountIgnoreChannel_)) {
        ResetDetectResult();
        return true;
    }
    return false;
}
```

## 7. OpenSL ES适配层 (OpenSL ES Adaptation)

### 7.1 音频播放器适配
**文件**: `frameworks/native/opensles/src/adapter/audioplayer_adapter.cpp` (第249-354行)
```cpp
// PCM格式转换
void AudioPlayerAdapter::ConvertPcmFormat(SLDataFormat_PCM *slFormat, AudioRendererParams *rendererParams)
{
    AudioSampleFormat sampleFormat = SlToOhosSampelFormat(slFormat);
    AudioSamplingRate sampleRate = SlToOhosSamplingRate(slFormat);
    AudioChannel channelCount = SlToOhosChannel(slFormat);
    rendererParams->sampleFormat = sampleFormat;
    rendererParams->sampleRate = sampleRate;
    rendererParams->channelCount = channelCount;
    rendererParams->encodingType = ENCODING_PCM;
}

// OpenSL ES采样格式到OHOS格式转换
AudioSampleFormat AudioPlayerAdapter::SlToOhosSampelFormat(SLDataFormat_PCM *pcmFormat)
{
    AudioSampleFormat sampleFormat;
    switch (pcmFormat->bitsPerSample) {
        case SL_PCMSAMPLEFORMAT_FIXED_8:
            sampleFormat = SAMPLE_U8;
            break;
        case SL_PCMSAMPLEFORMAT_FIXED_16:
            sampleFormat = SAMPLE_S16LE;
            break;
        case SL_PCMSAMPLEFORMAT_FIXED_24:
            sampleFormat = SAMPLE_S24LE;
            break;
        case SL_PCMSAMPLEFORMAT_FIXED_32:
            sampleFormat = SAMPLE_S32LE;
            break;
        default:
            sampleFormat = INVALID_WIDTH;
    }
    return sampleFormat;
}

// 声道数转换
AudioChannel AudioPlayerAdapter::SlToOhosChannel(SLDataFormat_PCM *pcmFormat)
{
    AudioChannel channelCount;
    switch (pcmFormat->numChannels) {
        case MONO:
            channelCount = MONO;
            break;
        case STEREO:
            channelCount = STEREO;
            break;
        default:
            channelCount = MONO;
            AUDIO_ERR_LOG("AudioPlayerAdapter::channel count not supported ");
    }
    return channelCount;
}
```

### 7.2 音频捕获器适配
**文件**: `frameworks/native/opensles/src/adapter/audiocapturer_adapter.cpp` (第205-310行)
```cpp
void AudioCapturerAdapter::ConvertPcmFormat(SLDataFormat_PCM *slFormat, AudioCapturerParams *capturerParams)
{
    AudioSampleFormat sampleFormat = SlToOhosSampelFormat(slFormat);
    AudioSamplingRate sampleRate = SlToOhosSamplingRate(slFormat);
    AudioChannel channelCount = SlToOhosChannel(slFormat);
    capturerParams->audioSampleFormat = sampleFormat;
    capturerParams->samplingRate = sampleRate;
    capturerParams->audioChannel = channelCount;
    capturerParams->audioEncoding = ENCODING_PCM;
}
```

## 8. 硬件抽象层适配器 (Hardware Abstraction Layer Adapters)

### 8.1 音频输出Sink适配器
**目录**: `frameworks/native/hdiadapter_new/sink/`

不同输出类型的适配器：
- `audio_render_sink.cpp` - 标准音频输出
- `fast_audio_render_sink.cpp` - 快速音频输出  
- `direct_audio_render_sink.cpp` - 直通音频输出
- `multichannel_audio_render_sink.cpp` - 多声道音频输出
- `offload_audio_render_sink.cpp` - 卸载音频输出
- `bluetooth_audio_render_sink.cpp` - 蓝牙音频输出
- `remote_audio_render_sink.cpp` - 远程音频输出

### 8.2 音频输入Source适配器  
**目录**: `frameworks/native/hdiadapter_new/source/`

不同输入类型的适配器：
- `audio_capture_source.cpp` - 标准音频输入
- `fast_audio_capture_source.cpp` - 快速音频输入
- `bluetooth_audio_capture_source.cpp` - 蓝牙音频输入
- `remote_audio_capture_source.cpp` - 远程音频输入
- `wakeup_audio_capture_source.cpp` - 唤醒音频输入

## 9. 关键数据结构和常量

### 9.1 音频参数常量
**文件**: `services/audio_service/server/src/pro_renderer_stream_impl.cpp` (第29-41行)
```cpp
constexpr uint64_t AUDIO_NS_PER_S = 1000000000;           // 纳秒转换
constexpr int32_t SECOND_TO_MILLISECOND = 1000;           // 秒到毫秒转换
constexpr int32_t DEFAULT_BUFFER_MILLISECOND = 20;        // 默认缓冲区时长(ms)
constexpr int32_t DEFAULT_BUFFER_MICROSECOND = 20000000;  // 默认缓冲区时长(μs)
constexpr uint32_t DOUBLE_VALUE = 2;                      // 双倍值
constexpr int32_t DEFAULT_RESAMPLE_QUANTITY = 2;          // 默认重采样数量
constexpr int32_t STEREO_CHANNEL_COUNT = 2;               // 立体声声道数
constexpr int32_t DEFAULT_TOTAL_SPAN_COUNT = 2;           // 默认总跨度数
constexpr int32_t DRAIN_WAIT_TIMEOUT_TIME = 100;          // 排空等待超时时间
constexpr int32_t FIRST_FRAME_TIMEOUT_TIME = 500;         // 首帧超时时间
```

### 9.2 流类型映射
**文件**: `frameworks/native/audiorenderer/src/audio_renderer.cpp` (第63-82行)
```cpp
static const std::map<AudioStreamType, StreamUsage> STREAM_TYPE_USAGE_MAP = {
    {STREAM_MUSIC, STREAM_USAGE_MUSIC},                    // 音乐
    {STREAM_VOICE_CALL, STREAM_USAGE_VOICE_COMMUNICATION}, // 语音通话
    {STREAM_VOICE_CALL_ASSISTANT, STREAM_USAGE_VOICE_CALL_ASSISTANT}, // 语音助手通话
    {STREAM_VOICE_ASSISTANT, STREAM_USAGE_VOICE_ASSISTANT}, // 语音助手
    {STREAM_ALARM, STREAM_USAGE_ALARM},                    // 闹钟
    {STREAM_VOICE_MESSAGE, STREAM_USAGE_VOICE_MESSAGE},    // 语音消息
    {STREAM_RING, STREAM_USAGE_RINGTONE},                  // 铃声
    {STREAM_NOTIFICATION, STREAM_USAGE_NOTIFICATION},      // 通知
    {STREAM_ACCESSIBILITY, STREAM_USAGE_ACCESSIBILITY},    // 无障碍
    {STREAM_SYSTEM, STREAM_USAGE_SYSTEM},                  // 系统音效
    {STREAM_MOVIE, STREAM_USAGE_MOVIE},                    // 电影
    {STREAM_GAME, STREAM_USAGE_GAME},                      // 游戏
    {STREAM_SPEECH, STREAM_USAGE_AUDIOBOOK},               // 语音/有声书
    {STREAM_NAVIGATION, STREAM_USAGE_NAVIGATION},          // 导航
    {STREAM_DTMF, STREAM_USAGE_DTMF},                      // 双音多频
    {STREAM_SYSTEM_ENFORCED, STREAM_USAGE_ENFORCED_TONE},  // 系统强制音效
    {STREAM_ULTRASONIC, STREAM_USAGE_ULTRASONIC},          // 超声波
    {STREAM_VOICE_RING, STREAM_USAGE_VOICE_RINGTONE},      // 语音铃声
};
```

这个代码映射文档显示了音频数据在OpenHarmony音频框架中从应用层到硬件层的每个处理步骤的具体实现位置，为开发者提供了详细的代码参考。