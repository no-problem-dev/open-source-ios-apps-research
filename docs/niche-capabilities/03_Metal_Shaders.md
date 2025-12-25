# Metal Shaderでカスタムビジュアル

**驚きポイント**: SwiftUIビューの背景にGPUで描画したリアルタイムエフェクト

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [Filterpedia](https://github.com/FlexMonkey/Filterpedia) | 2,317 | Core Image全フィルタカタログ |
| [Atmos](https://github.com/dejager/atmos) | 244 | SwiftUI + Metal雨エフェクト |
| [Mesh Transform Animation](https://github.com/jtrivedi/Mesh-Transform-Animation) | 239 | Dynamic Island風変形 |
| [TriangleDraw](https://github.com/triangledraw/TriangleDraw-iOS) | 62 | Metal描画 + Apple Pencil |

---

## 1. Metal + SwiftUI統合

### MTKViewをSwiftUIでラップ

```swift
import SwiftUI
import MetalKit

struct MetalView: UIViewRepresentable {
    let device: MTLDevice?
    let renderer: MetalRenderer

    func makeUIView(context: Context) -> MTKView {
        let view = MTKView()
        view.device = device
        view.delegate = renderer
        view.framebufferOnly = false
        view.clearColor = MTLClearColor(red: 0, green: 0, blue: 0, alpha: 0)
        view.isOpaque = false  // 背景透明
        return view
    }

    func updateUIView(_ uiView: MTKView, context: Context) {}
}

// SwiftUIで使用
struct ContentView: View {
    @StateObject private var renderer = MetalRenderer()

    var body: some View {
        ZStack {
            MetalView(
                device: MTLCreateSystemDefaultDevice(),
                renderer: renderer
            )

            VStack {
                Text("Metal背景")
                    .font(.largeTitle)
            }
        }
    }
}
```

---

## 2. シンプルなFragment Shader

### 雨エフェクト（Atmos風）

```metal
// rain.metal
#include <metal_stdlib>
using namespace metal;

struct VertexOut {
    float4 position [[position]];
    float2 uv;
};

// 雨粒シェーダー
fragment float4 rainFragment(
    VertexOut in [[stage_in]],
    constant float &time [[buffer(0)]],
    constant float2 &resolution [[buffer(1)]]
) {
    float2 uv = in.uv;

    // 雨粒のパターン
    float rain = 0.0;
    for (int i = 0; i < 10; i++) {
        float fi = float(i);
        float2 offset = float2(
            sin(fi * 1.2 + time * 0.5) * 0.1,
            fmod(fi * 0.1 + time * (0.3 + fi * 0.05), 1.0)
        );

        float2 pos = fmod(uv + offset, 1.0);
        float drop = smoothstep(0.02, 0.0, length(pos - float2(0.5, 0.5)));
        rain += drop * 0.15;
    }

    return float4(0.7, 0.8, 1.0, rain);
}
```

### Swift側セットアップ

```swift
class MetalRenderer: NSObject, ObservableObject, MTKViewDelegate {
    private var device: MTLDevice!
    private var commandQueue: MTLCommandQueue!
    private var pipelineState: MTLRenderPipelineState!
    private var startTime: Date = Date()

    override init() {
        super.init()
        setupMetal()
    }

    private func setupMetal() {
        device = MTLCreateSystemDefaultDevice()
        commandQueue = device.makeCommandQueue()

        let library = device.makeDefaultLibrary()
        let vertexFunction = library?.makeFunction(name: "vertexShader")
        let fragmentFunction = library?.makeFunction(name: "rainFragment")

        let descriptor = MTLRenderPipelineDescriptor()
        descriptor.vertexFunction = vertexFunction
        descriptor.fragmentFunction = fragmentFunction
        descriptor.colorAttachments[0].pixelFormat = .bgra8Unorm

        // アルファブレンド有効化
        descriptor.colorAttachments[0].isBlendingEnabled = true
        descriptor.colorAttachments[0].sourceRGBBlendFactor = .sourceAlpha
        descriptor.colorAttachments[0].destinationRGBBlendFactor = .oneMinusSourceAlpha

        pipelineState = try? device.makeRenderPipelineState(descriptor: descriptor)
    }

    func draw(in view: MTKView) {
        guard let drawable = view.currentDrawable,
              let descriptor = view.currentRenderPassDescriptor else { return }

        var time = Float(Date().timeIntervalSince(startTime))
        var resolution = float2(Float(view.drawableSize.width), Float(view.drawableSize.height))

        let commandBuffer = commandQueue.makeCommandBuffer()
        let encoder = commandBuffer?.makeRenderCommandEncoder(descriptor: descriptor)

        encoder?.setRenderPipelineState(pipelineState)
        encoder?.setFragmentBytes(&time, length: MemoryLayout<Float>.size, index: 0)
        encoder?.setFragmentBytes(&resolution, length: MemoryLayout<float2>.size, index: 1)
        encoder?.drawPrimitives(type: .triangleStrip, vertexStart: 0, vertexCount: 4)

        encoder?.endEncoding()
        commandBuffer?.present(drawable)
        commandBuffer?.commit()
    }
}
```

---

## 3. Mesh Transform（Dynamic Island風）

### メッシュ変形シェーダー

```metal
// mesh_transform.metal
struct Vertex {
    float2 position;
    float2 texCoord;
};

vertex VertexOut meshVertex(
    uint vertexID [[vertex_id]],
    constant Vertex *vertices [[buffer(0)]],
    constant float &morphFactor [[buffer(1)]]
) {
    Vertex v = vertices[vertexID];

    // モーフィング計算
    float2 morphedPos = v.position;

    // 中央に向かって収縮
    float2 center = float2(0.5, 0.5);
    float2 toCenter = center - v.position;
    morphedPos += toCenter * morphFactor * 0.3;

    // 縦方向に圧縮
    morphedPos.y = mix(morphedPos.y, center.y, morphFactor * 0.5);

    VertexOut out;
    out.position = float4(morphedPos * 2.0 - 1.0, 0.0, 1.0);
    out.uv = v.texCoord;
    return out;
}
```

### グリッドメッシュ生成

```swift
struct MeshGrid {
    let rows: Int
    let cols: Int
    var vertices: [SIMD2<Float>]
    var indices: [UInt16]

    init(rows: Int, cols: Int) {
        self.rows = rows
        self.cols = cols
        self.vertices = []
        self.indices = []

        // 頂点生成
        for row in 0...rows {
            for col in 0...cols {
                let x = Float(col) / Float(cols)
                let y = Float(row) / Float(rows)
                vertices.append(SIMD2<Float>(x, y))
            }
        }

        // インデックス生成（三角形ストリップ）
        for row in 0..<rows {
            for col in 0..<cols {
                let topLeft = UInt16(row * (cols + 1) + col)
                let topRight = topLeft + 1
                let bottomLeft = UInt16((row + 1) * (cols + 1) + col)
                let bottomRight = bottomLeft + 1

                indices.append(contentsOf: [
                    topLeft, bottomLeft, topRight,
                    topRight, bottomLeft, bottomRight
                ])
            }
        }
    }
}
```

---

## 4. Core Imageカスタムフィルタ

### CIKernel（Metal版）

```metal
// custom_filter.ci.metal
#include <CoreImage/CoreImage.h>

extern "C" {
    float4 pixelateFilter(
        coreimage::sampler src,
        float pixelSize
    ) {
        float2 uv = src.coord();

        // ピクセル化
        float2 pixelated = floor(uv / pixelSize) * pixelSize;

        return src.sample(pixelated);
    }
}
```

### Swift側

```swift
import CoreImage

class PixelateFilter: CIFilter {
    @objc dynamic var inputImage: CIImage?
    @objc dynamic var pixelSize: CGFloat = 10

    private static var kernel: CIKernel? = {
        guard let url = Bundle.main.url(
            forResource: "custom_filter",
            withExtension: "ci.metallib"
        ),
        let data = try? Data(contentsOf: url) else {
            return nil
        }
        return try? CIKernel(
            functionName: "pixelateFilter",
            fromMetalLibraryData: data
        )
    }()

    override var outputImage: CIImage? {
        guard let input = inputImage,
              let kernel = Self.kernel else { return nil }

        return kernel.apply(
            extent: input.extent,
            roiCallback: { _, rect in rect },
            arguments: [input, pixelSize]
        )
    }
}
```

---

## 5. Compute Shader（GPU計算）

### 画像処理の並列化

```metal
// compute.metal
kernel void grayscale(
    texture2d<float, access::read> inTexture [[texture(0)]],
    texture2d<float, access::write> outTexture [[texture(1)]],
    uint2 gid [[thread_position_in_grid]]
) {
    if (gid.x >= inTexture.get_width() || gid.y >= inTexture.get_height()) {
        return;
    }

    float4 color = inTexture.read(gid);
    float gray = dot(color.rgb, float3(0.299, 0.587, 0.114));
    outTexture.write(float4(gray, gray, gray, color.a), gid);
}
```

### Swift側

```swift
class ComputeProcessor {
    private let device: MTLDevice
    private let commandQueue: MTLCommandQueue
    private let pipelineState: MTLComputePipelineState

    init() {
        device = MTLCreateSystemDefaultDevice()!
        commandQueue = device.makeCommandQueue()!

        let library = device.makeDefaultLibrary()!
        let function = library.makeFunction(name: "grayscale")!
        pipelineState = try! device.makeComputePipelineState(function: function)
    }

    func process(_ texture: MTLTexture) -> MTLTexture {
        let descriptor = MTLTextureDescriptor.texture2DDescriptor(
            pixelFormat: texture.pixelFormat,
            width: texture.width,
            height: texture.height,
            mipmapped: false
        )
        descriptor.usage = [.shaderRead, .shaderWrite]
        let output = device.makeTexture(descriptor: descriptor)!

        let commandBuffer = commandQueue.makeCommandBuffer()!
        let encoder = commandBuffer.makeComputeCommandEncoder()!

        encoder.setComputePipelineState(pipelineState)
        encoder.setTexture(texture, index: 0)
        encoder.setTexture(output, index: 1)

        let threadGroupSize = MTLSize(width: 16, height: 16, depth: 1)
        let threadGroups = MTLSize(
            width: (texture.width + 15) / 16,
            height: (texture.height + 15) / 16,
            depth: 1
        )
        encoder.dispatchThreadgroups(threadGroups, threadsPerThreadgroup: threadGroupSize)

        encoder.endEncoding()
        commandBuffer.commit()
        commandBuffer.waitUntilCompleted()

        return output
    }
}
```

---

## まとめ

| 技術 | 用途 | 難易度 |
|------|------|--------|
| Fragment Shader | 背景エフェクト | 中 |
| Mesh Transform | 形状アニメーション | 高 |
| CIKernel | 画像フィルタ | 中 |
| Compute Shader | 並列GPU計算 | 高 |

### 実装のヒント

1. **デバッグ**: Xcode GPU Frame Capture
2. **パフォーマンス**: Metal System Trace
3. **互換性**: MTLDevice.supportsFamily()でチェック
4. **SwiftUI統合**: UIViewRepresentableでMTKViewをラップ
