# オーディオ合成・MIDI

**驚きポイント**: iPhoneでプロ級シンセサイザーを実装

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [AudioKitSynthOne](https://github.com/AudioKit/AudioKitSynthOne) | 1,751 | フル機能シンセ |
| [Monotone Delay](https://github.com/jkandzi/Monotone-Delay) | 18 | シンプルなシンセ |

---

## 1. AudioKit基礎

### セットアップ

```swift
import AudioKit
import AudioKitEX
import SoundpipeAudioKit

class SynthEngine: ObservableObject {
    let engine = AudioEngine()
    var oscillator: Oscillator!
    var envelope: AmplitudeEnvelope!

    init() {
        oscillator = Oscillator(waveform: Table(.sine))
        envelope = AmplitudeEnvelope(oscillator)

        // エンベロープ設定
        envelope.attackDuration = 0.01
        envelope.decayDuration = 0.1
        envelope.sustainLevel = 0.5
        envelope.releaseDuration = 0.3

        engine.output = envelope
        try? engine.start()
    }

    func noteOn(frequency: Float) {
        oscillator.frequency = frequency
        oscillator.amplitude = 1.0
        oscillator.start()
        envelope.openGate()
    }

    func noteOff() {
        envelope.closeGate()
    }
}
```

---

## 2. 波形生成

```swift
class WaveformGenerator {
    // サイン波
    func sine(frequency: Float, sampleRate: Float, samples: Int) -> [Float] {
        (0..<samples).map { i in
            sin(2 * .pi * frequency * Float(i) / sampleRate)
        }
    }

    // 矩形波
    func square(frequency: Float, sampleRate: Float, samples: Int) -> [Float] {
        (0..<samples).map { i in
            let t = Float(i) / sampleRate
            let phase = fmod(t * frequency, 1.0)
            return phase < 0.5 ? 1.0 : -1.0
        }
    }

    // ノコギリ波
    func sawtooth(frequency: Float, sampleRate: Float, samples: Int) -> [Float] {
        (0..<samples).map { i in
            let t = Float(i) / sampleRate
            let phase = fmod(t * frequency, 1.0)
            return 2.0 * phase - 1.0
        }
    }

    // 三角波
    func triangle(frequency: Float, sampleRate: Float, samples: Int) -> [Float] {
        (0..<samples).map { i in
            let t = Float(i) / sampleRate
            let phase = fmod(t * frequency, 1.0)
            return phase < 0.5
                ? 4.0 * phase - 1.0
                : 3.0 - 4.0 * phase
        }
    }
}
```

---

## 3. Core Audio低レベルAPI

```swift
import AudioToolbox
import AVFoundation

class CoreAudioSynth {
    var audioUnit: AudioComponentInstance?

    init() {
        var desc = AudioComponentDescription(
            componentType: kAudioUnitType_Output,
            componentSubType: kAudioUnitSubType_RemoteIO,
            componentManufacturer: kAudioUnitManufacturer_Apple,
            componentFlags: 0,
            componentFlagsMask: 0
        )

        let component = AudioComponentFindNext(nil, &desc)!
        AudioComponentInstanceNew(component, &audioUnit)

        // 出力フォーマット設定
        var streamFormat = AudioStreamBasicDescription(
            mSampleRate: 44100,
            mFormatID: kAudioFormatLinearPCM,
            mFormatFlags: kAudioFormatFlagIsFloat | kAudioFormatFlagIsPacked,
            mBytesPerPacket: 4,
            mFramesPerPacket: 1,
            mBytesPerFrame: 4,
            mChannelsPerFrame: 1,
            mBitsPerChannel: 32,
            mReserved: 0
        )

        AudioUnitSetProperty(
            audioUnit!,
            kAudioUnitProperty_StreamFormat,
            kAudioUnitScope_Input,
            0,
            &streamFormat,
            UInt32(MemoryLayout<AudioStreamBasicDescription>.size)
        )

        // レンダーコールバック設定
        var callbackStruct = AURenderCallbackStruct(
            inputProc: renderCallback,
            inputProcRefCon: Unmanaged.passUnretained(self).toOpaque()
        )

        AudioUnitSetProperty(
            audioUnit!,
            kAudioUnitProperty_SetRenderCallback,
            kAudioUnitScope_Input,
            0,
            &callbackStruct,
            UInt32(MemoryLayout<AURenderCallbackStruct>.size)
        )

        AudioUnitInitialize(audioUnit!)
    }

    func start() {
        AudioOutputUnitStart(audioUnit!)
    }

    func stop() {
        AudioOutputUnitStop(audioUnit!)
    }
}

// レンダーコールバック
func renderCallback(
    inRefCon: UnsafeMutableRawPointer,
    ioActionFlags: UnsafeMutablePointer<AudioUnitRenderActionFlags>,
    inTimeStamp: UnsafePointer<AudioTimeStamp>,
    inBusNumber: UInt32,
    inNumberFrames: UInt32,
    ioData: UnsafeMutablePointer<AudioBufferList>?
) -> OSStatus {
    let bufferList = UnsafeMutableAudioBufferListPointer(ioData)
    let buffer = bufferList![0]
    let samples = buffer.mData!.assumingMemoryBound(to: Float.self)

    // サイン波生成
    static var phase: Float = 0
    let frequency: Float = 440
    let sampleRate: Float = 44100
    let phaseIncrement = frequency / sampleRate

    for i in 0..<Int(inNumberFrames) {
        samples[i] = sin(2 * .pi * phase)
        phase += phaseIncrement
        if phase >= 1.0 { phase -= 1.0 }
    }

    return noErr
}
```

---

## 4. MIDI入力

```swift
import CoreMIDI

class MIDIReceiver {
    var midiClient: MIDIClientRef = 0
    var inputPort: MIDIPortRef = 0

    weak var delegate: MIDIReceiverDelegate?

    init() {
        MIDIClientCreate("MIDIClient" as CFString, nil, nil, &midiClient)

        MIDIInputPortCreate(
            midiClient,
            "Input" as CFString,
            midiReadProc,
            Unmanaged.passUnretained(self).toOpaque(),
            &inputPort
        )

        // すべてのMIDIソースに接続
        for i in 0..<MIDIGetNumberOfSources() {
            let source = MIDIGetSource(i)
            MIDIPortConnectSource(inputPort, source, nil)
        }
    }
}

protocol MIDIReceiverDelegate: AnyObject {
    func noteOn(note: UInt8, velocity: UInt8)
    func noteOff(note: UInt8)
    func controlChange(controller: UInt8, value: UInt8)
}

func midiReadProc(
    pktlist: UnsafePointer<MIDIPacketList>,
    readProcRefCon: UnsafeMutableRawPointer?,
    srcConnRefCon: UnsafeMutableRawPointer?
) {
    let receiver = Unmanaged<MIDIReceiver>.fromOpaque(readProcRefCon!).takeUnretainedValue()

    var packet = pktlist.pointee.packet
    for _ in 0..<pktlist.pointee.numPackets {
        let status = packet.data.0 & 0xF0
        let note = packet.data.1
        let velocity = packet.data.2

        switch status {
        case 0x90: // Note On
            if velocity > 0 {
                receiver.delegate?.noteOn(note: note, velocity: velocity)
            } else {
                receiver.delegate?.noteOff(note: note)
            }
        case 0x80: // Note Off
            receiver.delegate?.noteOff(note: note)
        case 0xB0: // Control Change
            receiver.delegate?.controlChange(controller: note, value: velocity)
        default:
            break
        }

        packet = MIDIPacketNext(&packet).pointee
    }
}
```

---

## 5. エフェクト

```swift
import AudioKit
import SoundpipeAudioKit

class EffectsChain {
    let reverb: Reverb
    let delay: Delay
    let filter: LowPassFilter

    init(input: Node) {
        // ローパスフィルター
        filter = LowPassFilter(input)
        filter.cutoffFrequency = 5000

        // ディレイ
        delay = Delay(filter)
        delay.time = 0.3
        delay.feedback = 0.5
        delay.dryWetMix = 0.3

        // リバーブ
        reverb = Reverb(delay)
        reverb.dryWetMix = 0.4
    }

    var output: Node { reverb }
}
```

---

## 6. MIDIノート変換

```swift
struct MIDIHelper {
    // ノート番号 → 周波数
    static func noteToFrequency(_ note: UInt8) -> Float {
        440.0 * pow(2.0, (Float(note) - 69.0) / 12.0)
    }

    // 周波数 → ノート番号
    static func frequencyToNote(_ frequency: Float) -> UInt8 {
        UInt8(69 + 12 * log2(frequency / 440.0))
    }

    // ノート番号 → 名前
    static func noteName(_ note: UInt8) -> String {
        let names = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"]
        let octave = Int(note / 12) - 1
        let name = names[Int(note % 12)]
        return "\(name)\(octave)"
    }
}
```

---

## まとめ

| 技術 | 用途 | 難易度 |
|------|------|--------|
| AudioKit | 高レベルシンセ | 低 |
| Core Audio | 低レベル制御 | 高 |
| CoreMIDI | MIDI入出力 | 中 |
| AVAudioEngine | エフェクトチェーン | 中 |

### 学習リソース

- [AudioKit Documentation](https://audiokit.io)
- [Core Audio Overview](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/CoreAudioOverview/)
