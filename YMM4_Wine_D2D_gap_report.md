# YMM4 × Wine d2d1 ギャップ分析レポート

対象: **YukkuriMovieMaker4 v4.56.1.0** (.NET 10 / Vortice.Direct2D1 / D3D11) vs **Wine master `dlls/d2d1/`** (18ファイル 25,729行)
手法: YMM4全14アセンブリのdnfileによるMemberRef走査(361+件のD2D呼び出し抽出) ↔ Wine d2d1ソースのスタブ/実装状況の全数照合。行番号はwine-d2d1/内の該当ソース。

---

## 結論(要約)

Wineのd2d1は「描画プリミティブとリソース管理」はほぼ完成しているが、**D2Dの心臓部であるエフェクトグラフのGPU実行パスが存在しない**。YMM4は全アイテムの描画をD2Dエフェクトチェーン(ぼかし・色調補正・クロマキー等)で行うため、エフェクトが1つでも乗った要素が**一切描画されない**。これが最大のブロッカー。加えて、YMM4が使う32種の内蔵エフェクトのうち**20種が未登録**(CreateEffectが失敗)、コマンドリストが**記録も再生も失敗**する。

---

## P0: 致命的ブロッカー(これが無いとYMM4の描画パイプラインが成立しない)

### P0-1. エフェクト出力の描画が存在しない — `DrawImage(ID2D1Image)`
- **YMM4用途**: 全アイテムのエフェクトチェーン描画。`ID2D1Effect.get_Output` → `DeviceContext.DrawImage(effect)` がYMM4の中心描画パス。
- **Wine実装**: `d2d_device_context_DrawImage` (device.c:2716) は `QueryInterface(IID_ID2D1Bitmap)` が成功する画像しか描画しない。エフェクト画像(effect.c:3050 `d2d_effect_image_vtbl`)やコマンドリストは `FIXME("Unhandled image %p.")` (device.c:2752) で**何も描かずにreturn**。
- **影響**: エフェクト適用済みアイテムが全滅。Opacity/Composite等の合成エフェクト経由の描画も全滅。プレビュー・書き出し両方が黒/空。
- **必要な実装**: エフェクトグラフの実際のD3D11実行(入力テクスチャ→ピクセルシェーダーパス→中間ターゲット→合成)。Wineのeffect.cはプロパティ管理・シェーダー読み込み(LoadPixelShader等, effect.c:2471/2498/2525)まで実装済みだが、**実行・ラスタライズ部分が無い**。d2d1全体で最大の未完成領域。

### P0-2. コマンドリストが記録も再生も失敗
- **YMM4用途**: `CreateCommandList` / `set_Target` / `Close` / `get_Target`(YukkuriMovieMaker.dll)。
- **Wine実装**:
  - コマンドリストをターゲットにしたまま `EndDraw` → **E_NOTIMPL** (device.c:2076 "Unimplemented for command list target")。記録の確定ができない。
  - `ID2D1CommandList::Close` は状態フラップのみ、ターゲット解除すらしない (command_list.c:505 "FIXME: reset as a target")。
  - 再生は `ID2D1CommandSink` を**呼び出し元が提供する**場合のみ機能 (command_list.c:297以降)。Wine自身はCommandSinkを実装しないため、自前のレンダーターゲットへのコマンドリスト再生が不可能。`DrawImage(commandList)` もP0-1同様 "Unhandled image"。
- **影響**: コマンドリストを使うキャッシュ/合成パスが全滅。

### P0-3. 内蔵エフェクト20種が未登録 — `CreateEffect` が失敗
- **YMM4が使用しWineに存在しないCLSID** (effect.c:1935-1954の登録表との差分):
  `AlphaMask, ChromaKey, CrossFade, DistantDiffuse, DistantSpecular, EdgeDetection, Exposure, GammaTransfer, HighlightsAndShadows, Invert, LinearTransfer, LuminanceToAlpha, Morphology, Opacity, PointDiffuse, Posterize, Sepia, Sharpen, Tile, Tint` — **20種**
- **Wine登録済み20種のうちYMM4が使うもの**: 2DAffineTransform, 3DPerspectiveTransform, 3DTransform, Composite, Crop, Shadow, Grayscale, ColorMatrix, Flood, GaussianBlur, PointSpecular, ArithmeticComposite, Blend, Brightness, DirectionalBlur, HueRotation, Saturation, Scale, Premultiply, ColorManagement
- **影響**: クロマキー(ChromaKey)・セピア(Sepia)・ポスタライズ(Posterize)・シャープ(Sharpen)・タイル(Tile)等、YMM4の「エフェクト」UIの大半が `CreateEffect` 時点で失敗。
- **必要な実装**: 各エフェクトのHLSL+プロパティ定義(Windows SDKのd2d1effects.h準拠)。Wine側は既存20種のパターン(effect.cのdescriptionテーブル)に追加するだけの機械的作業だが、P0-1が解消されないと意味がない。

### P0-4. エフェクト画像のバウンディングボックスが取得不能
- **YMM4用途**: `GetImageLocalBounds` / `GetImageWorldBounds`(アイテム配置・レイアウト計算)。
- **Wine実装**: `GetImageWorldBounds` は**常にE_NOTIMPL** (device.c:2472)。`GetImageLocalBounds` はビットマップのみ対応、それ以外はE_NOTIMPL (device.c:2463)。
- **影響**: エフェクト出力の座標計算が失敗し、レイアウトロジックが破綻(例外または0x0扱い)。

### P0-5. カスタムエフェクト初期化の2点
- `ID2D1EffectContext::GetMaximumSupportedFeatureLevel` → **E_NOTIMPL** (effect.c:2426)。YMM4のCustomEffectBaseは初期化時にこれを呼ぶため、カスタムエフェクト(ぷるぷる等のHLSL自作エフェクト)の生成パスが失敗し得る。
- `ID2D1DrawInfo::SetVertexProcessing` → **no-opスタブ** (effect.c:3448付近, FIXME)。頂点シェーダーでメッシュ変形するエフェクト(YMM4のLoadVertexShader+CreateVertexBuffer+SetVertexProcessing使用)が機能しない。
- ※登録(`RegisterEffectFromStream` factory.c:1046)、シェーダー読み込み(effect.c:2471/2498/2525)、頂点バッファ(effect.c:2589)、プロパティシステム、`SetSingleTransformNode`(effect.c:749)は**実装済み**で、この2点とP0-1が残る。

---

## P1: 機能単位で死ぬもの(起動はするが対応機能が無効化)

### P1-1. カラーグリフ(絵文字/COLRフォント)
- `DrawColorBitmapGlyphRun` (device.c:3117) / `DrawSvgGlyphRun` (device.c:3125) は**voidスタブ**。
- dwrite側は `TranslateColorGlyphRun` (main.c:1786, IDWriteFactory4版)・`IDWriteColorGlyphRunEnumerator1` (font.c:6230) が実装済みなので、**D2D側の描画だけ**が欠けている。絵文字入テキストが白/透明になる。

### P1-2. SVG (FileSource.Svg プラグイン)
- `CreateSvgDocument` E_NOTIMPL (device.c:3168)、`DrawSvgDocument` voidスタブ (device.c:3171)、`CreateSvgGlyphStyle` E_NOTIMPL (device.c:3094)、`GetColorBitmapGlyphImage`/`GetSvgGlyphImage` E_NOTIMPL (device.c:3144/3159)。ID2D1SvgDocument/SvgElement API一式が無い → **SVGアイテム読み込みは完全に死んでいる**。

### P1-3. インク & グラデーションメッシュ
- `CreateInk` (device.c:2949) / `CreateInkStyle` (:2957) / `DrawInk` (:3008) / `CreateGradientMesh` (:2967) / `DrawGradientMesh` (:3014) / `GetGradientMeshWorldBounds` (:3005) — 全部スタブ。
- **d2d1.spec に `D2D1GetGradientMeshInteriorPointsFromCoonsPatch` のエクスポートが無い**(d2d1.specは12エクスポートのみ)→ Vortice経由の呼び出しで `EntryPointNotFoundException`。

### P1-4. その他のDeviceContextスタブ
- `BlendImage` (Context6, device.c:3661) voidスタブ。
- `CreateColorContext` / `CreateColorContextFromFilename` / `CreateColorContextFromWicColorContext` (device.c:2208/2216/2224) E_NOTIMPL、`ID2D1Bitmap1::GetColorContext` (bitmap.c:252) スタブ → ICCプロファイル埋め込みPNG/JPEG出力(FileWriter.PngWav)が退化。
- `ID2D1DeviceContext::CreateGradientStopCollection`(カラー補間指定版, device.c:2333) E_NOTIMPL ※YMM4は旧RenderTarget版(実装済み)を使用のため影響小。
- `ID2D1Properties::SetCached` no-op(キャッシュ効果なし、性能のみ)。

### P1-5. ジオメトリ演算 (geometry.c、E_NOTIMPL 42箇所の全マップ)
全ジオメトリタイプ共通で死んでいるもの(YMM4が使用):
- `CombineWithGeometry`: path(:4417) / ellipse(:4936) / rect(:5387) / rounded(:5789) / transformed(:6191) — **全タイプスタブ**
- `GetWidenedBounds`: path(:4030) / ellipse(:4814) / rect(:5220) / rounded(:5671) / transformed(:6102) — 全タイプスタブ
- `Widen`: path(:4462) / ellipse(:4982) / rect(:5452) / rounded(:5835) / transformed(:6242) — 全タイプスタブ
- `GetBounds`: ellipse(:4804) / rounded(:5661) のみスタブ(path/rect/group/transformedは実装済み)
- `ComputeLength` / `ComputePointAtLength` / `Outline` / `CompareWithGeometry` / `ComputePointAndSegmentAtLength`(pathのみ, :4693) — 各タイプスタブ
- **影響**: 図形の輪郭処理・結合・拡大線幅計算を使う機能(立ち絵のクリップ領域、図形エフェクトの当たり計算等)が失敗。基本描画(DrawGeometry/FillGeometry)とパス構築(GeometrySink一式)は実装済みなので、単純な図形描画は生きる。

---

## 実装済みでYMM4が問題なく使えるもの(照合済み)

| 分類 | 内容 |
|---|---|
| ターゲット基本 | BeginDraw/EndDraw(ビットマップターゲット時)/Clear/Flush、set/get_Target、set_PrimitiveBlend(device.c:2649)、PushLayer/PopLayer(device.c:1833)、アンチエイリアス/トランスフォーム/DPI/MaximumBitmapSize |
| 描画プリミティブ | DrawBitmap/DrawImage(ビットマップのみ)/DrawLine/DrawRectangle/FillRectangle/DrawEllipse/FillEllipse/FillRoundedRectangle/DrawGeometry/FillGeometry/DrawText/DrawTextLayout/DrawGlyphRun(IDWriteTextRenderer実装 device.c:3606) |
| ブラシ | SolidColorBrush/LinearGradient/RadialGradient/BitmapBrush/ImageBrush(ビットマップソースのみ)/GradientStopCollection(RenderTarget版) |
| ビットマップ | CreateBitmap/CreateBitmapFromDxgiSurface(device.c:2267)/CreateBitmapFromWicBitmap(device.c:384)/Map/Unmap/GetSurface(bitmap.c:264) |
| ジオメトリ構築 | CreatePathGeometry/Rectangle/RoundedRectangle/Ellipse/GeometryGroup/TransformedGeometry + GeometrySink一式 + **GeometryRealization**(CreateFilledGeometryRealization/DrawGeometryRealization, geometry.c:6748) |
| ファクトリ/デバイス | ID2D1Factory7(ID2D1Factory1のRegisterEffect/GetRegisteredEffects/UnregisterEffect含む)/ID2D1Device6/CreateDeviceContext/ClearResources(device.c:4658)/DesktopDpi/Multithread Enter/Leave(factory.c:1330) |
| エフェクト基盤 | CreateEffect(登録済みCLSIDのみ)/プロパティシステム(Get*/SetValue)/SetInput/InputCount/Output/カスタムエフェクト登録・シェーダー読み込み・頂点バッファ |
| dwrite(別DLL) | IDWriteFactory7(=4/6互換)/TranslateColorGlyphRun(main.c:1786)/IDWriteFontFace5(=3互換)/IDWriteFontSet+Builder2/IDWriteTextLayout4/ColorGlyphRunEnumerator1 — **dwriteはほぼブロッカーなし** |
| D2D1エクスポート | D2D1CreateFactory等12関数(※GradientMeshInteriorPointsのみ欠落) |

---

## Wine側に足すべき実装の優先順位(実装作業見積り付き)

1. **エフェクトグラフの実行パス**(P0-1): d2d_effectに「入力→シェーダーパス→出力テクスチャ」のラスタライズを追加し、DrawImage/DrawBitmap/CommandSinkがエフェクト画像を描けるようにする。最大・最難関(数千行規模)。ここが解けないとYMM4は何も描かない。
2. **コマンドリストの確定と自己再生**(P0-2): EndDraw(command list target)の実装、Closeでのターゲット解除、Wine内部用ID2D1CommandSink実装。
3. **欠落エフェクト20種の登録**(P0-3): 既存パターンへの追加。各エフェクトHLSLはWindows SDK `d2d1effects.h`+`d2d1effecthelpers.hlsli`から移植可能。比較的機械的。
4. **GetImageWorldBounds/GetImageLocalBounds**(P0-4): エフェクト未実装の間はビットマップと同様の暫定計算でもYMM4のレイアウトは前進する。
5. **GetMaximumSupportedFeatureLevel + SetVertexProcessing**(P0-5): 前者はD3D11フィーチャーレベルの返却のみの軽実装。後者は頂点シェーダーパスで本格実装が必要。
6. **DrawColorBitmapGlyphRun/DrawSvgGlyphRun**(P1-1): dwriteのTranslateColorGlyphRun結果をD2Dビットマップグリフとして描く。絵文字表示に必須。
7. **ジオメトリ演算**(P1-5): GetWidenedBounds/Widen/CombineWithGeometry等。flattening+ブール演算が必要で中規模。
8. **SVG/Ink/GradientMesh/ColorContext**(P1-2〜4): YMM4の該当プラグイン・機能単位で個別に。

---

## 補足(スコープ外だが重要)

- **WPF**: YMM4のUIはWPF(PresentationCore参照あり)。.NET 10のWPFがWineで動くかはD2D以前の更大な壁であり、本レポートの対象外。D2Dは動画プレビュー・書き出しパス。
- **DXGI**: YMM4は`IDXGIKeyedMutex`(AcquireSync/ReleaseSync)と`IDXGIResource1::CreateSharedHandle`で共有サーフェイス(エンコーダ連携)を使う。WineのDXGI実装状況は未検証(要動作確認)。
- **WIC**: デコード/エンコード/FormatConverter/PaletteはWineのwindowscodecsでほぼ動くと推定されるが、ColorContext(ICC)周りは要確認。
- 本分析の根拠データ: `d2d_usage_report.txt`(YMM4側全呼び出し1074行)、`extract_d2d_usage.py`(抽出スクリプト)、`wine-d2d1/`・`wine-dwrite/`(Wine masterソース)。
