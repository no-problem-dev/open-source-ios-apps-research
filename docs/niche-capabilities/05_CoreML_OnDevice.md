# CoreMLオンデバイス推論

**驚きポイント**: サーバー不要でリアルタイム画像認識・音声認識

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [Find](https://github.com/aheze/OpenFind) | 1,045 | 画像内テキスト検索 |
| [SeeFood](https://github.com/kingreza/SeeFood) | 448 | 料理認識 |
| [Awesome ML](https://github.com/eugenebokhan/Awesome-ML) | 229 | モデル管理UI |
| [AlohaGIF](https://github.com/michaello/Aloha) | 64 | 音声→字幕GIF |

---

## 1. Vision + CoreML

### 画像分類

```swift
import Vision
import CoreML

class ImageClassifier {
    private lazy var model: VNCoreMLModel? = {
        guard let mlModel = try? MobileNetV2(configuration: .init()).model else { return nil }
        return try? VNCoreMLModel(for: mlModel)
    }()

    func classify(_ image: CGImage) async throws -> [(label: String, confidence: Float)] {
        guard let model = model else { throw ClassifierError.modelNotLoaded }

        return try await withCheckedThrowingContinuation { continuation in
            let request = VNCoreMLRequest(model: model) { request, error in
                if let error = error {
                    continuation.resume(throwing: error)
                    return
                }

                let results = (request.results as? [VNClassificationObservation]) ?? []
                let classifications = results.prefix(5).map {
                    (label: $0.identifier, confidence: $0.confidence)
                }
                continuation.resume(returning: classifications)
            }

            let handler = VNImageRequestHandler(cgImage: image)
            try? handler.perform([request])
        }
    }
}
```

### 物体検出

```swift
class ObjectDetector {
    private lazy var model: VNCoreMLModel? = {
        guard let mlModel = try? YOLOv3(configuration: .init()).model else { return nil }
        return try? VNCoreMLModel(for: mlModel)
    }()

    struct Detection {
        let label: String
        let confidence: Float
        let boundingBox: CGRect  // 正規化座標 (0-1)
    }

    func detect(in image: CGImage) async throws -> [Detection] {
        guard let model = model else { throw DetectorError.modelNotLoaded }

        return try await withCheckedThrowingContinuation { continuation in
            let request = VNCoreMLRequest(model: model) { request, error in
                let results = (request.results as? [VNRecognizedObjectObservation]) ?? []

                let detections = results.compactMap { observation -> Detection? in
                    guard let label = observation.labels.first else { return nil }
                    return Detection(
                        label: label.identifier,
                        confidence: label.confidence,
                        boundingBox: observation.boundingBox
                    )
                }
                continuation.resume(returning: detections)
            }

            let handler = VNImageRequestHandler(cgImage: image)
            try? handler.perform([request])
        }
    }
}
```

---

## 2. テキスト認識（OCR）

```swift
import Vision

class TextRecognizer {
    func recognizeText(in image: CGImage) async throws -> [String] {
        try await withCheckedThrowingContinuation { continuation in
            let request = VNRecognizeTextRequest { request, error in
                let observations = request.results as? [VNRecognizedTextObservation] ?? []

                let texts = observations.compactMap { observation in
                    observation.topCandidates(1).first?.string
                }
                continuation.resume(returning: texts)
            }

            // 日本語対応
            request.recognitionLanguages = ["ja-JP", "en-US"]
            request.recognitionLevel = .accurate

            let handler = VNImageRequestHandler(cgImage: image)
            try? handler.perform([request])
        }
    }

    // 画像内テキスト検索（Find風）
    func searchText(_ query: String, in image: CGImage) async throws -> [CGRect] {
        let texts = try await recognizeText(in: image)
        // マッチする領域を返す
        return []  // 実装省略
    }
}
```

---

## 3. 音声認識

```swift
import Speech

class SpeechRecognizer: ObservableObject {
    private let recognizer = SFSpeechRecognizer(locale: Locale(identifier: "ja-JP"))
    private var recognitionTask: SFSpeechRecognitionTask?
    private let audioEngine = AVAudioEngine()

    @Published var transcription: String = ""
    @Published var isRecording = false

    func requestAuthorization() async -> Bool {
        await withCheckedContinuation { continuation in
            SFSpeechRecognizer.requestAuthorization { status in
                continuation.resume(returning: status == .authorized)
            }
        }
    }

    func startRecording() throws {
        let request = SFSpeechAudioBufferRecognitionRequest()
        request.shouldReportPartialResults = true

        let inputNode = audioEngine.inputNode
        let recordingFormat = inputNode.outputFormat(forBus: 0)

        inputNode.installTap(onBus: 0, bufferSize: 1024, format: recordingFormat) { buffer, _ in
            request.append(buffer)
        }

        recognitionTask = recognizer?.recognitionTask(with: request) { [weak self] result, error in
            if let result = result {
                DispatchQueue.main.async {
                    self?.transcription = result.bestTranscription.formattedString
                }
            }
        }

        audioEngine.prepare()
        try audioEngine.start()
        isRecording = true
    }

    func stopRecording() {
        audioEngine.stop()
        audioEngine.inputNode.removeTap(onBus: 0)
        recognitionTask?.cancel()
        isRecording = false
    }
}
```

---

## 4. カスタムモデル

### Create MLでトレーニング

```swift
import CreateML

// 画像分類モデルのトレーニング（macOS）
func trainImageClassifier() throws {
    let trainingData = MLImageClassifier.DataSource.labeledDirectories(
        at: URL(fileURLWithPath: "/path/to/training")
    )

    let classifier = try MLImageClassifier(
        trainingData: trainingData,
        parameters: MLImageClassifier.ModelParameters(
            maxIterations: 20,
            augmentationOptions: [.crop, .rotation]
        )
    )

    try classifier.write(to: URL(fileURLWithPath: "MyModel.mlmodel"))
}
```

### オンデバイスでモデル更新

```swift
import CoreML

class ModelUpdater {
    func updateModel(with newData: MLBatchProvider) async throws -> MLModel {
        let modelURL = Bundle.main.url(forResource: "UpdatableModel", withExtension: "mlmodelc")!

        let updateTask = try MLUpdateTask(
            forModelAt: modelURL,
            trainingData: newData,
            configuration: nil,
            completionHandler: { _ in }
        )

        updateTask.resume()

        // 更新完了を待機
        return try await withCheckedThrowingContinuation { continuation in
            updateTask.completionHandler = { context in
                if let error = context.task.error {
                    continuation.resume(throwing: error)
                } else {
                    let updatedModel = try! MLModel(contentsOf: context.model.modelURL)
                    continuation.resume(returning: updatedModel)
                }
            }
        }
    }
}
```

---

## 5. パフォーマンス最適化

### GPU/Neural Engine活用

```swift
let config = MLModelConfiguration()
config.computeUnits = .all  // CPU + GPU + Neural Engine

let model = try MyModel(configuration: config)
```

### バッチ処理

```swift
class BatchProcessor {
    func processImages(_ images: [CGImage]) async throws -> [[String: Float]] {
        let model = try MobileNetV2(configuration: .init())

        // バッチ入力を作成
        let inputs = try images.map { image in
            try MobileNetV2Input(imageWith: image)
        }

        let batchProvider = MLArrayBatchProvider(array: inputs)
        let results = try model.model.predictions(fromBatch: batchProvider)

        return (0..<results.count).map { index in
            let output = results.features(at: index)
            // 結果を辞書に変換
            return [:]  // 実装省略
        }
    }
}
```

---

## まとめ

| 機能 | フレームワーク | 用途 |
|------|---------------|------|
| 画像分類 | Vision + CoreML | 物体識別 |
| 物体検出 | Vision + CoreML | 位置特定 |
| OCR | Vision | テキスト抽出 |
| 音声認識 | Speech | 文字起こし |
| カスタムML | Create ML | 独自モデル |

### 実装のヒント

1. **モデル選択**: 精度 vs サイズ vs 速度のトレードオフ
2. **量子化**: Float16/Int8で軽量化
3. **Neural Engine**: A11以降で高速推論
4. **バックグラウンド**: BGProcessingTaskで長時間処理
