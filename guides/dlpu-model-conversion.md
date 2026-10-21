# Preparing Custom Models for the DLPU: From PyTorch to an INT8 Model, Per Chip

> **Documented, with reported and inferred claims labelled.** Everything below comes from Axis's
> developer documentation, the AxisCommunications GitHub repositories, and (where noted) upstream
> tools' documentation and source code. Claims that rest only on a GitHub issue are marked
> **Reported** with the year; reasoning that no source states outright is marked **Inference**.
> Checked against Axis's documentation as published in October 2026, when ACAP Native SDK 12.11 for
> AXIS OS 12.11 was the current release ([ACAP releases](https://www.axis.com/en-us/for-developers/news/ACAP-releases)).
> Nothing here was re-tested on a device for this guide. Sources are linked inline and listed under
> [References](#references).

## The short answer

Training in PyTorch is fine — Axis's conversion pages name PyTorch and TensorFlow as the common
training options ([ARTPEC-8 model conversion](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-8/model-conversion/)) —
but Axis documents no way to run a PyTorch file on the camera. Each accelerator takes a chip-specific
artifact ([Axis DLPU overview](https://developer.axis.com/computer-vision/on-device/axis-dlpu/)):

- **ARTPEC-7, ARTPEC-8, ARTPEC-9:** a *quantized* TensorFlow Lite (`.tflite`) model. ARTPEC-7 also
  needs an Edge TPU compile.
- **CV25, CV75 (Ambarella):** a native `.bin` compiled by Ambarella's toolchain — from ONNX (CV25 can
  also start from `.tflite`) ([CV25](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv25/model-conversion/)
  and [CV75](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv75/model-conversion/)
  model conversion).

On the ARTPEC chips the quantization must be **int8/uint8**: Axis's optimization tips for ARTPEC-7,
ARTPEC-8 and ARTPEC-9 each say operators with other data types are not supported on the DLPU
([ARTPEC-7](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-7/optimization-tips/),
[ARTPEC-8](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-8/optimization-tips/),
[ARTPEC-9](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-9/optimization-tips/)).
Axis documents no FP16 path for these accelerators. In Axis's conversion table every ARTPEC row
includes a quantization step, and the CPU is the only TensorFlow Lite target that takes a plain,
unquantized `.tflite` ([Axis DLPU overview](https://developer.axis.com/computer-vision/on-device/axis-dlpu/)).
**Reported (2023):** an fp16 YOLOv7 `.tflite` failed to load on ARTPEC-8
([issue #140](https://github.com/AxisCommunications/acap-computer-vision-sdk-examples/issues/140)).
Axis puts the DLPU at more than 100× the CPU's speed for many models
([DLPU vs CPU](https://developer.axis.com/computer-vision/on-device/cpu-vs-dlpu/)).

**Inference:** if someone asks "8x8 or FP16?", the answer for ARTPEC is full-integer 8-bit — 8-bit
weights *and* 8-bit activations. The Axis pages cited here don't use the term "8x8"; the reading
comes from Google's naming of its "16x8" mode ([LiteRT post-training quantization](https://developers.google.com/edge/litert/conversion/tensorflow/quantization/post_training_quantization)).

## Per-chip requirements

| | ARTPEC-7 | ARTPEC-8 | ARTPEC-9 | CV25 | CV75 |
|---|---|---|---|---|---|
| You hand over | quantized `.tflite`, compiled with `edgetpu_compiler` | quantized `.tflite` | quantized `.tflite` | Ambarella `.bin` | Ambarella `.bin` |
| Who quantizes | you | you | you | Ambarella's toolchain | Ambarella's toolchain |
| Precision on the accelerator | int8/uint8 | int8/uint8 | int8/uint8 | not published | not published |
| Granularity | — | **per-tensor** | per-channel allowed only for weights/bias of some operators | not public | not public |
| Larod device | `google-edge-tpu-tflite` | `axis-a8-dlpu-tflite` | `a9-dlpu-tflite` | `ambarella-cvflow` | `cv75-dlpu-native` |

Sources: per-chip [model conversion](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-8/model-conversion/)
and [optimization tips](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-9/optimization-tips/)
pages (same URL pattern for each chip); the
[CV25](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv25/model-conversion/) and
[CV75](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv75/model-conversion/)
conversion pages; the [ONNX to CV25](https://developer.axis.com/computer-vision/on-device/how-to-guides/onnx-to-cv25/)
and [ONNX to CV75](https://developer.axis.com/computer-vision/on-device/how-to-guides/onnx-to-cv75/)
guides, where the toolchain generates the quantization data; the
[tensorflow-to-larod](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/tensorflow-to-larod)
example for the `edgetpu_compiler` step; and [Larod NN inference](https://developer.axis.com/acap/acap-native-sdk-version-12/api/src/api/larod/html/md__opt_builder-doc_larod_doc_nn-inference.html)
for the device names. Not every ARTPEC-7 product has an Edge TPU
([ARTPEC-7 model conversion](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-7/model-conversion/));
run `larod-client -a` on the device to list what is actually there
([larod-client](https://developer.axis.com/acap/api/src/api/larod/html/md_doc_larod-client.html)).

## PyTorch → INT8 TFLite (ARTPEC-7/8/9)

Axis's [PyTorch to TFLite guide](https://developer.axis.com/computer-vision/on-device/how-to-guides/pytorch-to-tflite/)
converts in three hops:

1. `torch.onnx.export(...)` with `opset_version=13` and a static input shape (`[1, 3, 224, 224]` in
   the guide, no `dynamic_axes`).
2. `onnx2tf.convert()` to a TensorFlow SavedModel.
3. `tf.lite.TFLiteConverter.from_saved_model()` with full-integer quantization.

The guide pins the versions it was tested with: torch 2.13.0, tensorflow 2.19.0, numpy 2.4.6,
pillow 12.2.0, onnx2tf 1.29.24 and onnxscript 0.7.1. Its converter settings, condensed (the script's
defaults are `QUANTIZATION_DTYPE = "int8"` and `QUANTIZATION_METHOD = "per_channel"`):

```python
converter = tf.lite.TFLiteConverter.from_saved_model(onnx2tf_output_dir)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = lambda: representative_dataset_generator(input_shape=INPUT_SHAPE)
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8     # tf.uint8 when QUANTIZATION_DTYPE == "uint8"
converter.inference_output_type = tf.int8    # tf.uint8 when QUANTIZATION_DTYPE == "uint8"
if QUANTIZATION_METHOD == "per_tensor":
    converter._experimental_disable_per_channel = True
```

**The ARTPEC-8 trap:** Axis's PyTorch and TensorFlow how-to scripts expose the granularity as
`QUANTIZATION_METHOD` and **default it to `"per_channel"`**
([PyTorch guide](https://developer.axis.com/computer-vision/on-device/how-to-guides/pytorch-to-tflite/);
[TensorFlow guide](https://developer.axis.com/computer-vision/on-device/how-to-guides/tensorflow-to-tflite/)),
while the ARTPEC-8 pages call for per-tensor and say it performs much better than per-channel
([ARTPEC-8 model conversion](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-8/model-conversion/);
[ARTPEC-8 tips](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-8/optimization-tips/)).
Set it to `"per_tensor"` for ARTPEC-8. **Reported (2025):** after per-tensor quantization, one
user's MobileNetV1-variant model ran about 10× faster on an ARTPEC-8 camera but returned the same
output for every input; a MobileNetV2 variant converted the same way seemed to work
([issue #360](https://github.com/AxisCommunications/acap-native-sdk-examples/issues/360)) — check
accuracy, not just speed.

**Calibration data.** Use roughly 100–200 varied images from the camera's real scene. The guide's
generator resizes each image, converts to float32 and divides by 255 — no mean/std normalization
([PyTorch guide](https://developer.axis.com/computer-vision/on-device/how-to-guides/pytorch-to-tflite/)).
**Inference:** if your training pipeline normalizes with a mean/std, move that normalization inside
the exported network, so calibration sees the same 0–1 range that the model's 8-bit input (raw
camera pixel values) is quantized to represent.

**uint8 vs int8.** Axis's guides offer both. Axis's
[`tensorflow-to-larod` conversion script](https://github.com/AxisCommunications/acap-native-sdk-examples/blob/main/tensorflow-to-larod/env/convert_model.py)
sets uint8 input and output, and the upstream
[YOLOv5 export](https://github.com/ultralytics/yolov5/blob/95ebf68f92196975e53ebc7e971d0130432ad107/export.py)
that Axis's YOLOv5 workflow uses does the same. The
[vdo-larod](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/vdo-larod) example
feeds frames, scaled and colour-converted by Larod preprocessing, into a uint8 input tensor
([`model.c`](https://github.com/AxisCommunications/acap-native-sdk-examples/blob/main/vdo-larod/app/model.c)).
**Inference:** uint8 I/O lets those frames go in without an extra conversion step.

**ARTPEC-7 extra step.** Compile the quantized model for the Edge TPU, as in Axis's example:
`edgetpu_compiler -s -o <out_dir> <model>.tflite`
([tensorflow-to-larod](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/tensorflow-to-larod)).
Coral's documentation says `-s` prints which operations mapped to the Edge TPU
([Coral Edge TPU compiler](https://gweb-coral-full.uc.r.appspot.com/docs/edgetpu/compiler/)).

**Validation is on you.** The Axis guide ends when the `.tflite` file exists; it contains no
float-vs-int8 accuracy check
([PyTorch guide](https://developer.axis.com/computer-vision/on-device/how-to-guides/pytorch-to-tflite/)).
Axis's testing page covers copying the model to the camera, running it with `larod-client`, and reading
`journalctl -u larod` and the system log
([Test your model](https://developer.axis.com/computer-vision/on-device/test-your-model/)).

### YOLO models

For YOLOv5, Axis documents a separate path: a pinned YOLOv5 commit plus an Axis patch, exported with
`export.py --include tflite --int8 --per-tensor` for ARTPEC-8 and `--int8` for ARTPEC-9
([YOLOv5 on Axis cameras](https://github.com/AxisCommunications/axis-model-zoo/blob/main/docs/yolov5.md)).
Both patches set the activation to `nn.ReLU6()`
([ARTPEC-8 patch](https://acap-ml-models.s3.amazonaws.com/yolov5/yolov5_artpec8.patch);
[ARTPEC-9 patch](https://acap-ml-models.s3.amazonaws.com/yolov5/yolov5_artpec9.patch)), overriding
YOLOv5's default SiLU
([`common.py`](https://github.com/ultralytics/yolov5/blob/95ebf68f92196975e53ebc7e971d0130432ad107/models/common.py)
and [`yolo.py`](https://github.com/ultralytics/yolov5/blob/95ebf68f92196975e53ebc7e971d0130432ad107/models/yolo.py)
at the pinned commit).

Axis documents no YOLOv8/YOLO11 path for ARTPEC-8 or ARTPEC-9; its Model Zoo lists YOLOv8n only for
CV75 ([Axis Model Zoo](https://github.com/AxisCommunications/axis-model-zoo)). Since Ultralytics
8.4.83, `format='tflite'` redirects to a LiteRT export
([`exporter.py`](https://github.com/ultralytics/ultralytics/blob/main/ultralytics/engine/exporter.py)),
whose INT8 mode keeps float32 graph input and output
([`litert.py`](https://github.com/ultralytics/ultralytics/blob/main/ultralytics/utils/export/litert.py);
both checked at Ultralytics 8.4.183), and the `litert` export exposes no per-tensor option.
**Inference:** that does not match the int8/uint8-I/O, per-tensor model the ARTPEC-8 pages call for,
so treat it as unvalidated on Axis DLPUs.

## Ambarella CV25 / CV75

- **Toolchain:** Ambarella's CNNGen toolchain is not publicly available; Axis says to contact
  Ambarella directly ([CV25 model conversion](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv25/model-conversion/);
  [CV75 model conversion](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv75/model-conversion/)).
- **Input:** CV25 accepts `.tflite` only with CNNGen 2.7 or earlier, and Axis documents only an ONNX
  path for CV75 (same sources). You supply an ONNX model plus representative images, which the
  toolchain uses for quantization
  ([ONNX to CV25](https://developer.axis.com/computer-vision/on-device/how-to-guides/onnx-to-cv25/);
  [ONNX to CV75](https://developer.axis.com/computer-vision/on-device/how-to-guides/onnx-to-cv75/)).
- **Steps Axis documents** (inside the CNNGen toolchain container, about 100–200 calibration images)
  ([ONNX to CV25](https://developer.axis.com/computer-vision/on-device/how-to-guides/onnx-to-cv25/);
  [ONNX to CV75](https://developer.axis.com/computer-vision/on-device/how-to-guides/onnx-to-cv75/)):

  ```bash
  gen_image_list.py -f <img_dir> -o img_list.txt -ns -e .jpg -c 0 -d 0,0 -r <H>,<W> \
    -bf dra_image_bin -bo dra_image_bin/dra_bin_list.txt
  onnxparser.py -m <model>.onnx \
    -isrc "is:1,3,<H>,<W>|iq|idf:0,0,8,0|i:<input_name>=dra_image_bin/dra_bin_list.txt" \
    -o quantized -of quantized -odst "o:<output_name>|odf:fp32"
  vas -auto -show-progress quantized/quantized.vas
  cavalry_gen -d vas_output/ -f <model>_cv25.bin    # <model>_cv75.bin for CV75
  ```

  Axis notes that these steps must be adapted to each model.

- **Not public:** the supported-ops and optimization-tips pages for CV25 and CV75 say the information
  is not public ([CV25 supported ops](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv25/supported-ops/);
  [CV25 tips](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv25/optimization-tips/);
  [CV75 supported ops](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv75/supported-ops/);
  [CV75 tips](https://developer.axis.com/computer-vision/on-device/system-on-chip/cv75/optimization-tips/)).
- **CV25 constraints from Axis's example:** the input row width must be a multiple of 32 (the example
  pads 300-pixel rows to 320; the height stays 300), and the accelerator cannot do bounding-box
  post-processing, so decoding and NMS run in your app
  ([object-detection-cv25](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/object-detection-cv25)).

## Per-chip design rules and optimization tips

| Chip | Documented rules and recommendations |
|---|---|
| ARTPEC-7 | The Edge TPU compiler can split a model only once: the first unsupported operator and everything after it run on the CPU — avoid unsupported operators, especially early in the graph ([ARTPEC-7 tips](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-7/optimization-tips/)) |
| ARTPEC-8 | Prefer regular 3×3 convolutions over depthwise, use its native stride 2, filter counts in multiples of 6, and ReLU rather than LeakyReLU ([ARTPEC-8 tips](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-8/optimization-tips/)) |
| ARTPEC-9 | Batch size 1; kernel sizes from {1, 2, 3, 5, 7, 9}; channel counts in multiples of 16; prefer ReLU6; avoid explicit padding ([ARTPEC-9 supported ops](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-9/supported-ops/); [ARTPEC-9 tips](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-9/optimization-tips/)) |

- **SiLU** does not appear in the ARTPEC-8 or ARTPEC-9 supported-ops lists
  ([ARTPEC-8](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-8/supported-ops/);
  [ARTPEC-9](https://developer.axis.com/computer-vision/on-device/system-on-chip/artpec-9/supported-ops/)),
  and Axis's YOLOv5 patches replace it with ReLU6 (see above).
- **CPU offloading:** each switch between DLPU and CPU copies memory; repeated switching makes a model
  very slow, while offloading only the first or last operators may cost little
  ([DLPU vs CPU](https://developer.axis.com/computer-vision/on-device/cpu-vs-dlpu/)).
- **Reported (2023):** a YOLOv8m exported with Ultralytics `int8=True` failed on an ARTPEC-8 camera
  (AXIS OS 11.3) with `Model contains too many graph partitions (137 > 16)`. An Axis maintainer found
  in `journalctl -u larod` (`hybrid data type is not supported in conv2d`) that the model's conv2d
  layers were only weight-quantized and that unquantized Add/Mul operators forced CPU fallback
  ([issue #141](https://github.com/AxisCommunications/acap-computer-vision-sdk-examples/issues/141)).
- **Not documented:** how the ARTPEC-8 and ARTPEC-9 backends handle an unsupported operator (CPU split,
  rejection, or whole-model fallback), and any ARTPEC-8/9 equivalent of `edgetpu_compiler -s`.
  **Inference:** compare `larod-client -R` timings on `cpu-tflite` and on the DLPU device — a small
  speed-up suggests much of the model is running on the CPU.

## Running and testing on the device

Smoke-test a converted model with random input, as the model conversion pages show:

```bash
larod-client -g <model>.tflite -c axis-a8-dlpu-tflite -i ""   # ARTPEC-8
larod-client -g <model>.tflite -c a9-dlpu-tflite -i ""        # ARTPEC-9
larod-client -g <model>.bin -c ambarella-cvflow -i ""         # CV25
journalctl -u larod                                           # load errors land here
```

`larod-client -a` lists devices, `-R <n>` repeats jobs and reports the mean job time, and `-o` writes
outputs to a file ([larod-client](https://developer.axis.com/acap/api/src/api/larod/html/md_doc_larod-client.html)).

Facts from the [Larod NN inference](https://developer.axis.com/acap/acap-native-sdk-version-12/api/src/api/larod/html/md__opt_builder-doc_larod_doc_nn-inference.html)
documentation worth knowing before you ship:

- **ARTPEC-8** compiles a `.tflite` to its native format when the model is loaded, which can take
  minutes; the compiled-model cache is limited to 64 MiB and 16 files, oldest evicted first.
- **ARTPEC-9** also compiles at load time. Its backend uses Winograd convolution, which trades
  accuracy for speed; load with `disable-winograd:1` if accuracy suffers.
- **CV25/CV75** accept a cavalry `.bin` already compiled with Ambarella's toolchain (the docs describe
  no load-time compilation for them) and take no backend options; CV25 also accepts the deprecated
  `.larod` format.

App code selects the device by name with `larodGetDevice()` and loads the model with
`larodLoadModel()` ([Larod for app developers](https://developer.axis.com/acap/acap-native-sdk-version-12/api/src/api/larod/html/md__opt_builder-doc_larod_doc_introduction-for-app-developers.html)).
For the frame pipeline (VDO → Larod preprocessing → inference), start from
[vdo-larod](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/vdo-larod) and
[object-detection-yolov5](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/object-detection-yolov5).

## Packaging and deployment

- **One `.eap` per chip.** Axis's examples build a separate package per chip, bundling the model with
  `acap-build . -a 'model/model.tflite'`
  ([object-detection-yolov5](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/object-detection-yolov5)).
  ARTPEC-8, ARTPEC-9 and CV25 all build as `aarch64`
  ([Axis devices and compatibility](https://developer.axis.com/acap/axis-devices-and-compatibility/)).
  **Inference:** the package architecture therefore will not stop an ARTPEC-8 build from installing on
  an ARTPEC-9 or CV25 camera; it fails only at runtime, when the app asks Larod for a device the camera
  doesn't have (Axis's example manifests hard-code it in `runOptions`) or loads a model built for
  another chip.
- **Declare the DLPU.** On AXIS OS 13.0 an app that uses the DLPU must add a `deepLearningProcessor`
  object under `resources` in the manifest. Axis's example is `{"enabled": true, "required": true}`;
  `"required": false` lets the app install where DLPU access is unavailable
  ([Upgrade a manifest](https://developer.axis.com/acap/how-to-guides/upgrade-a-manifest/)).
- **Sign the package.** See [acap-application-signing.md](./acap-application-signing.md).
- **Mind the licence.** Axis's YOLOv5 patch and the YOLO models in its examples and Model Zoo are
  AGPL-3.0-only, following Ultralytics' terms
  ([YOLOv5 on Axis cameras](https://github.com/AxisCommunications/axis-model-zoo/blob/main/docs/yolov5.md);
  [Axis Model Zoo](https://github.com/AxisCommunications/axis-model-zoo);
  [object-detection-yolov5](https://github.com/AxisCommunications/acap-native-sdk-examples/tree/main/object-detection-yolov5)).
- **Share the DLPU carefully** — a bundled analytic loading its own model can starve yours; see
  [dlpu-memory-contention.md](./dlpu-memory-contention.md).

## Reference speeds

Mean inference times from the [Axis Model Zoo](https://github.com/AxisCommunications/axis-model-zoo),
measured on AXIS OS 12.11.72:

| Device (SoC) | MobileNetV2 | YOLOv5s |
|---|---|---|
| Q1615 Mk III (ARTPEC-7) | 4.42 ms (TF1 model) | — |
| P1465-LE (ARTPEC-8) | 9.89 ms | — |
| Q1656-LE (ARTPEC-8) | 5.36 ms | 69.97 ms |
| Q1728 (ARTPEC-9) | 2.02 ms | 35.00 ms |
| M3085-V (CV25) | 5.35 ms | 128.36 ms |
| M2048-LE (CV75) | 2.98 ms | — |

The Model Zoo lists a separate YOLOv5s variant per chip (ARTPEC-8, ARTPEC-9, CV25), so the YOLOv5s
column does not compare identical models.

The two ARTPEC-8 cameras differ by 1.8× on MobileNetV2, consistent with Axis's note that
speed depends on each product's clock and memory, not only the SoC
([Axis DLPU](https://developer.axis.com/computer-vision/on-device/axis-dlpu/)). Benchmark on the
camera model you will ship.

## Not documented (as of October 2026)

- What the ARTPEC-8/9 backends do with float, hybrid or unsupported-operator models, and any
  per-operator placement report for those chips.
- FP16 support on any ARTPEC accelerator.
- The precision CNNGen chooses for CV25/CV75, the CNNGen container image, and licence terms.
- Per-app DLPU memory quotas and maximum model or `.eap` size.

---

## References

- [Axis DLPU overview](https://developer.axis.com/computer-vision/on-device/axis-dlpu/) — conversion path per target (only the CPU takes an unquantized
  `.tflite`); DLPU type follows the SoC; speed also depends on each product's clock and memory.
- [DLPU vs CPU](https://developer.axis.com/computer-vision/on-device/cpu-vs-dlpu/) — >100× for many
  models; cost of CPU offloading.
- Per-chip pages (replace `<chip>` with `artpec-7`, `artpec-8`, `artpec-9`, `cv25`, `cv75`):
  `https://developer.axis.com/computer-vision/on-device/system-on-chip/<chip>/model-conversion/`,
  `.../optimization-tips/`, `.../supported-ops/` — int8/uint8 requirement, per-tensor on ARTPEC-8,
  ARTPEC-9 operator limits, the CNNGen notice for CV25/CV75.
- [PyTorch to TFLite](https://developer.axis.com/computer-vision/on-device/how-to-guides/pytorch-to-tflite/)
  and [TensorFlow to TFLite](https://developer.axis.com/computer-vision/on-device/how-to-guides/tensorflow-to-tflite/)
  — the export/convert/quantize recipe, pinned versions, `QUANTIZATION_METHOD` default, calibration.
- [ONNX to CV25](https://developer.axis.com/computer-vision/on-device/how-to-guides/onnx-to-cv25/) and
  [ONNX to CV75](https://developer.axis.com/computer-vision/on-device/how-to-guides/onnx-to-cv75/) — the
  CNNGen command sequence.
- [Test your model](https://developer.axis.com/computer-vision/on-device/test-your-model/) and
  [larod-client](https://developer.axis.com/acap/api/src/api/larod/html/md_doc_larod-client.html) —
  on-device testing.
- [Larod NN inference](https://developer.axis.com/acap/acap-native-sdk-version-12/api/src/api/larod/html/md__opt_builder-doc_larod_doc_nn-inference.html)
  and [Larod for app developers](https://developer.axis.com/acap/acap-native-sdk-version-12/api/src/api/larod/html/md__opt_builder-doc_larod_doc_introduction-for-app-developers.html)
  — device names, load-time compilation, ARTPEC-8 cache, Winograd option, device selection API.
- [Upgrade a manifest](https://developer.axis.com/acap/how-to-guides/upgrade-a-manifest/) — the
  `deepLearningProcessor` resource on AXIS OS 13.0.
- [Axis devices and compatibility](https://developer.axis.com/acap/axis-devices-and-compatibility/) —
  SDK architecture per SoC.
- [acap-native-sdk-examples](https://github.com/AxisCommunications/acap-native-sdk-examples) —
  `tensorflow-to-larod`, `vdo-larod`, `object-detection-yolov5`, `object-detection-cv25`.
- [ACAP releases](https://www.axis.com/en-us/for-developers/news/ACAP-releases) — ACAP Native SDK 12.11
  as the current release in October 2026.
- [Axis Model Zoo](https://github.com/AxisCommunications/axis-model-zoo) and its
  [YOLOv5 guide](https://github.com/AxisCommunications/axis-model-zoo/blob/main/docs/yolov5.md) —
  benchmarks, YOLOv5 export flags and patches, licensing.
- **Reported, not official:** GitHub issues
  [#140](https://github.com/AxisCommunications/acap-computer-vision-sdk-examples/issues/140),
  [#141](https://github.com/AxisCommunications/acap-computer-vision-sdk-examples/issues/141) (2023) and
  [#360](https://github.com/AxisCommunications/acap-native-sdk-examples/issues/360) (2025).
- **Upstream tools:** [LiteRT post-training quantization](https://developers.google.com/edge/litert/conversion/tensorflow/quantization/post_training_quantization),
  [Coral Edge TPU compiler](https://gweb-coral-full.uc.r.appspot.com/docs/edgetpu/compiler/),
  Ultralytics [`exporter.py`](https://github.com/ultralytics/ultralytics/blob/main/ultralytics/engine/exporter.py)
  and [`litert.py`](https://github.com/ultralytics/ultralytics/blob/main/ultralytics/utils/export/litert.py)
  (checked at 8.4.183), and YOLOv5 at the commit Axis pins
  ([`95ebf68`](https://github.com/ultralytics/yolov5/tree/95ebf68f92196975e53ebc7e971d0130432ad107):
  `export.py`, `models/common.py`, `models/yolo.py`).
