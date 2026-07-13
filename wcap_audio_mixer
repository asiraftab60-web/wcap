#pragma once

// Combines wcap's existing system-loopback capture (this already contains BOTH
// your game's audio AND Discord's audio, since Windows plays them through the
// same default output device) with a separate microphone capture, producing a
// single mixed float32/stereo/48kHz stream.
//
// Drop-in usage in wcap.c: replace `AudioCapture gAudio;` with `AudioMixer gAudio;`
// and swap the AudioCapture_* calls for the AudioMixer_* ones listed below.
// AudioMixerData has the same Samples/Count/Time layout as AudioCaptureData,
// so EncodeCapturedAudio() in wcap.c needs no other changes.
//
//   AudioMixer_Start(&gAudio, ApplicationWindow, gConfig.CaptureMicrophone, gConfig.MicrophoneGain)
//   AudioMixer_Flush(&gAudio)
//   AudioMixer_GetData(&gAudio, &Data, gEncoder.StartTime)
//   AudioMixer_ReleaseData(&gAudio, &Data)
//   AudioMixer_Stop(&gAudio)
//
// gAudio.Format is still valid (points at the system stream's WAVEFORMATEX)
// so `EncConfig.AudioFormat = gAudio.System.Format;` works unchanged.

#include "wcap.h"
#include "wcap_audio_capture.h"

//
// interface
//

typedef AudioCaptureData AudioMixerData;

typedef struct
{
    float*  Buf;            // interleaved stereo float32 ring
    uint32_t CapacityFrames; // power of two
    int64_t WriteCursor;    // frames written since stream start (monotonic)
    int64_t ReadCursor;     // frames consumed/retired so far (monotonic)
    uint64_t StartTime;     // QPC timestamp of frame 0 (Data->Time of first packet)
    bool    Started;
}
AudioMixerStage;

typedef struct
{
    AudioCapture    System; // game + Discord playback (loopback)
    AudioCapture    Mic;    // microphone
    bool            HasMic;
    float           MicGain;
    WAVEFORMATEX*   Format; // alias for System.Format, so existing `gAudio.Format` call sites in wcap.c need no changes
    uint64_t        Freq;   // alias for System.Freq, ditto

    AudioMixerStage SystemStage;
    AudioMixerStage MicStage;

    int64_t         MicOffsetFrames; // mic frame N corresponds to system frame (N + MicOffsetFrames)
    bool            MicOffsetComputed;

    float*          OutBuf;
    uint32_t        OutCapacityFrames;
}
AudioMixer;

// make sure CoInitializeEx has been called before calling Start()
// ApplicationWindow: pass a window handle to capture only that app's audio (see wcap's
//                    existing "application local audio" feature), or NULL for full system loopback
// CaptureMic: whether to also open and mix in the default microphone
// MicGain: linear gain applied to mic samples before mixing (1.0 = unity, e.g. 1.5 to boost a quiet mic)
static bool AudioMixer_Start(AudioMixer* Mixer, HWND ApplicationWindow, bool CaptureMic, float MicGain);
static void AudioMixer_Stop(AudioMixer* Mixer);
static void AudioMixer_Flush(AudioMixer* Mixer);
static bool AudioMixer_GetData(AudioMixer* Mixer, AudioMixerData* Data, uint64_t ExpectedTimestamp);
static void AudioMixer_ReleaseData(AudioMixer* Mixer, AudioMixerData* Data);

//
// implementation
//

static void AudioMixerStage__Init(AudioMixerStage* Stage, uint32_t CapacityFrames)
{
    Stage->Buf = (float*)VirtualAlloc(NULL, (SIZE_T)CapacityFrames * 2 * sizeof(float), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    Assert(Stage->Buf);
    Stage->CapacityFrames = CapacityFrames;
    Stage->WriteCursor = 0;
    Stage->ReadCursor = 0;
    Stage->StartTime = 0;
    Stage->Started = false;
}

static void AudioMixerStage__Free(AudioMixerStage* Stage)
{
    if (Stage->Buf)
    {
        VirtualFree(Stage->Buf, 0, MEM_RELEASE);
        Stage->Buf = NULL;
    }
}

// appends `Count` interleaved-stereo-float32 frames, assumed to be the next
// contiguous frames of a gapless stream (true for a single AudioCapture, since
// WASAPI reports silence gaps inline rather than as time gaps)
static void AudioMixerStage__Append(AudioMixerStage* Stage, const float* Samples, uint32_t Count, uint64_t Time)
{
    if (Count == 0)
    {
        return;
    }

    if (!Stage->Started)
    {
        Stage->StartTime = Time;
        Stage->WriteCursor = 0;
        Stage->ReadCursor = 0;
        Stage->Started = true;
    }

    // guard against ring overwrite if consumer ever falls too far behind
    int64_t Used = Stage->WriteCursor - Stage->ReadCursor;
    if (Used + (int64_t)Count > (int64_t)Stage->CapacityFrames)
    {
        Stage->ReadCursor = Stage->WriteCursor + (int64_t)Count - (int64_t)Stage->CapacityFrames;
    }

    uint32_t Mask = Stage->CapacityFrames - 1;
    for (uint32_t i = 0; i < Count; i++)
    {
        uint32_t Idx = (uint32_t)(Stage->WriteCursor + i) & Mask;
        Stage->Buf[Idx * 2 + 0] = Samples[i * 2 + 0];
        Stage->Buf[Idx * 2 + 1] = Samples[i * 2 + 1];
    }
    Stage->WriteCursor += Count;
}

static inline float AudioMixer__Clamp(float V)
{
    return V < -1.0f ? -1.0f : (V > 1.0f ? 1.0f : V);
}

bool AudioMixer_Start(AudioMixer* Mixer, HWND ApplicationWindow, bool CaptureMic, float MicGain)
{
    ZeroMemory(Mixer, sizeof(*Mixer));

    if (!AudioCapture_Start(&Mixer->System, ApplicationWindow))
    {
        return false;
    }

    Mixer->Format = Mixer->System.Format;
    Mixer->Freq   = Mixer->System.Freq;

    Mixer->MicGain = MicGain;
    Mixer->HasMic = false;

    if (CaptureMic)
    {
        // if the microphone can't be opened (no device, permission denied, etc)
        // we deliberately don't fail the whole recording - just record without mic
        Mixer->HasMic = AudioCapture_StartMicrophone(&Mixer->Mic);
    }

    // ~2.7 sec ring at 48kHz - comfortably larger than any expected mic/system start-time skew
    const uint32_t RingFrames = 1 << 17;
    AudioMixerStage__Init(&Mixer->SystemStage, RingFrames);
    if (Mixer->HasMic)
    {
        AudioMixerStage__Init(&Mixer->MicStage, RingFrames);
    }

    Mixer->OutCapacityFrames = RingFrames;
    Mixer->OutBuf = (float*)VirtualAlloc(NULL, (SIZE_T)Mixer->OutCapacityFrames * 2 * sizeof(float), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    Assert(Mixer->OutBuf);

    Mixer->MicOffsetComputed = false;

    return true;
}

void AudioMixer_Stop(AudioMixer* Mixer)
{
    AudioCapture_Stop(&Mixer->System);
    if (Mixer->HasMic)
    {
        AudioCapture_Stop(&Mixer->Mic);
        AudioMixerStage__Free(&Mixer->MicStage);
    }
    AudioMixerStage__Free(&Mixer->SystemStage);

    if (Mixer->OutBuf)
    {
        VirtualFree(Mixer->OutBuf, 0, MEM_RELEASE);
        Mixer->OutBuf = NULL;
    }
}

void AudioMixer_Flush(AudioMixer* Mixer)
{
    AudioCapture_Flush(&Mixer->System);
    if (Mixer->HasMic)
    {
        AudioCapture_Flush(&Mixer->Mic);
    }
}

// drains whatever WASAPI has produced on both streams into the staging rings
static void AudioMixer__Pump(AudioMixer* Mixer)
{
    AudioCaptureData Data;

    while (AudioCapture_GetData(&Mixer->System, &Data, 0))
    {
        AudioMixerStage__Append(&Mixer->SystemStage, (const float*)Data.Samples, (uint32_t)Data.Count, Data.Time);
        AudioCapture_ReleaseData(&Mixer->System, &Data);
    }

    if (Mixer->HasMic)
    {
        while (AudioCapture_GetData(&Mixer->Mic, &Data, 0))
        {
            AudioMixerStage__Append(&Mixer->MicStage, (const float*)Data.Samples, (uint32_t)Data.Count, Data.Time);
            AudioCapture_ReleaseData(&Mixer->Mic, &Data);
        }
    }

    if (Mixer->HasMic && !Mixer->MicOffsetComputed && Mixer->SystemStage.Started && Mixer->MicStage.Started)
    {
        uint64_t Freq = Mixer->System.Freq;
        uint32_t SampleRate = Mixer->System.Format->nSamplesPerSec;
        int64_t DeltaQpc = (int64_t)Mixer->MicStage.StartTime - (int64_t)Mixer->SystemStage.StartTime;
        // frames, in system-stream time, at which the mic stream's frame 0 occurred
        Mixer->MicOffsetFrames = MFllMulDiv(DeltaQpc, SampleRate, Freq, 0);
        Mixer->MicOffsetComputed = true;
    }
}

bool AudioMixer_GetData(AudioMixer* Mixer, AudioMixerData* Data, uint64_t ExpectedTimestamp)
{
    AudioMixer__Pump(Mixer);

    int64_t SystemAvail = Mixer->SystemStage.WriteCursor - Mixer->SystemStage.ReadCursor;
    if (SystemAvail <= 0)
    {
        return false;
    }

    uint32_t Frames = (uint32_t)min(SystemAvail, (int64_t)Mixer->OutCapacityFrames);
    uint32_t SysMask = Mixer->SystemStage.CapacityFrames - 1;

    bool MixMic = Mixer->HasMic && Mixer->MicOffsetComputed;
    uint32_t MicMask = MixMic ? (Mixer->MicStage.CapacityFrames - 1) : 0;

    for (uint32_t i = 0; i < Frames; i++)
    {
        int64_t SysFrame = Mixer->SystemStage.ReadCursor + i;
        uint32_t SysIdx = (uint32_t)SysFrame & SysMask;
        float L = Mixer->SystemStage.Buf[SysIdx * 2 + 0];
        float R = Mixer->SystemStage.Buf[SysIdx * 2 + 1];

        if (MixMic)
        {
            int64_t MicFrame = SysFrame - Mixer->MicOffsetFrames;
            if (MicFrame >= 0 && MicFrame < Mixer->MicStage.WriteCursor)
            {
                uint32_t MicIdx = (uint32_t)MicFrame & MicMask;
                L += Mixer->MicGain * Mixer->MicStage.Buf[MicIdx * 2 + 0];
                R += Mixer->MicGain * Mixer->MicStage.Buf[MicIdx * 2 + 1];
            }
            // else: mic has no data for this instant (not started yet / dropped out) -> system audio only
        }

        Mixer->OutBuf[i * 2 + 0] = AudioMixer__Clamp(L);
        Mixer->OutBuf[i * 2 + 1] = AudioMixer__Clamp(R);
    }

    Data->Samples = Mixer->OutBuf;
    Data->Count   = Frames;
    // same timestamp convention as AudioCapture_GetData: QPC ticks
    Data->Time = Mixer->SystemStage.StartTime + MFllMulDiv(Mixer->SystemStage.ReadCursor, Mixer->System.Freq, Mixer->System.Format->nSamplesPerSec, 0);

    Mixer->SystemStage.ReadCursor += Frames;

    if (MixMic)
    {
        int64_t ConsumedMicUpTo = Mixer->SystemStage.ReadCursor - Mixer->MicOffsetFrames;
        if (ConsumedMicUpTo > Mixer->MicStage.ReadCursor)
        {
            Mixer->MicStage.ReadCursor = ConsumedMicUpTo;
        }
    }

    (void)ExpectedTimestamp; // kept only for interface symmetry with AudioCapture_GetData

    return true;
}

void AudioMixer_ReleaseData(AudioMixer* Mixer, AudioMixerData* Data)
{
    // mixed samples are consumed immediately when produced in AudioMixer_GetData,
    // nothing to release - kept as a no-op so call sites mirror AudioCapture's pattern
    (void)Mixer;
    (void)Data;
}
