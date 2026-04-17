# Masks, dtypes, thresholding, and interpolation

Gotchas around binary masks when they flow through ANTs and SimpleITK.

## Use `GenericLabel` for segmentations, not `NearestNeighbor`

For discrete label maps / brain masks / segmentations, use `-n GenericLabel` with `antsApplyTransforms`, **not** `-n NearestNeighbor`.

`GenericLabel` was added to ITK specifically for this case. It evaluates a continuous interpolator (Linear by default) on each label's indicator function and picks the winning label per output voxel. The result has noticeably cleaner boundaries than NN and avoids NN's axis-aligned aliasing, especially at oblique resampling angles.

```
antsApplyTransforms -n GenericLabel ...                 # default inner interpolator = Linear
antsApplyTransforms -n GenericLabel[BSpline,3] ...      # optional: specify inner interpolator and param
```

Only fall back to `NearestNeighbor` when the label values themselves encode something continuous that you explicitly don't want blended (rare in practice).

For **intensity images**, interpolation choice is different:
- `Linear` for single-subject warping (reduces ringing at the brain boundary).
- `BSpline` for averaging multiple subjects (the averaging smooths any ringing and the BSpline's extra sharpness pays off).

## `antsApplyTransforms` writes float by default

Even when the input is `uint8` and interpolation is `GenericLabel`, the default output dtype is `float64`. You end up with a float mask whose nominal values are `0.0` and `1.0` but may have tiny numerical jitter. Pass `-u uchar` to force integer output:

```
antsApplyTransforms -d 3 -i mask.nii.gz -r ref.nii.gz -o out.nii.gz \
    -n GenericLabel -u uchar -t transform.mat
```

Output dtype options per ANTs docs: `char`, `uchar`, `short`, `int`, `float`, `double`, `default`. Use `uchar` for binary masks, `int` or `uint32` for multi-label CC images if you need > 255 labels.

Without `-u`, downstream tools may end up with a `float64` mask, and the next trap below waits.

## `sitk.BinaryThreshold` silently misbehaves on int pixel types with float thresholds

This is the nastiest gotcha in this skill. Calling:

```python
sitk.BinaryThreshold(int8_image, lowerThreshold=0.5, upperThreshold=1e10,
                     insideValue=1, outsideValue=0)
```

on an `int8` `{0, 1}` mask produces an **inverted** output — the 1s and 0s are swapped. Cause: the `1e10` upper bound is clipped to the `int8` range and the resulting "interval" collapses in a way that makes the filter assign the wrong label.

### Symptom

`LabelShapeStatisticsImageFilter.GetBoundingBox(1)` returns the full image extent and `GetNumberOfPixels(1)` matches the *background* voxel count. Downstream logic then treats the entire image as "foreground" and everything falls over.

### Fix

Always cast to float before thresholding a mask from unknown provenance:

```python
def binarize(image: sitk.Image) -> sitk.Image:
    """Threshold a labelmap-like image to {0, 1} uint8, robust to input dtype."""
    return sitk.BinaryThreshold(
        sitk.Cast(image, sitk.sitkFloat32),
        lowerThreshold=0.5, upperThreshold=1e10,
        insideValue=1, outsideValue=0,
    )
```

The float cast makes the upper bound not get clipped, and the cast is essentially free.

Alternative: use integer-friendly bounds for known-integer masks (e.g. `lowerThreshold=1, upperThreshold=1`), but the cast-to-float approach is more robust because it doesn't assume anything about the input value range.

## NRRD segmentation files from 3D Slicer

Slicer writes segmentations as `.seg.nrrd` files with extra metadata (segment colors, labels, extent offsets). `sitk.ReadImage` reads them as an ordinary image — the pixel values are typically `{0, 1}` for single-segment binary masks, or `{0, 1, 2, …}` for multi-segment. The segment metadata is preserved as SITK metadata keys like `Segment0_Color`, `Segment0_LabelValue`, `Segment0_Extent`.

For mask comparison work, treat `.seg.nrrd` files like any other labelmap: binarize (carefully — see above) and proceed.

## Checking a binarization worked

A quick sanity check after binarizing:

```python
arr = sitk.GetArrayFromImage(binarized)
print(f"dtype={arr.dtype}, min={arr.min()}, max={arr.max()}, foreground={int((arr != 0).sum())}")
```

Foreground count should match the known brain voxel count (~500k for a mouse brain at 100 µm).
