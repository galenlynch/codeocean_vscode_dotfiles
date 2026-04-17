# Command reference and options

Practical reference for `antsApplyTransforms` options and adjacent ANTs commands. See [transforms.md](transforms.md) for composition order and chain semantics.

## `antsApplyTransforms` — key options

```
antsApplyTransforms -d 3 \
  -i INPUT \             # image or labelmap to resample
  -r REFERENCE \         # defines output grid (size, spacing, direction, origin)
  -o OUTPUT \            # output path
  -n INTERPOLATION \     # Linear, BSpline[3], GenericLabel, NearestNeighbor, ...
  -u OUTPUT_TYPE \       # char, uchar, short, int, float, double, default
  -t TRANSFORM [-t ...]  # transform chain, leftmost applied first
```

Key options:

- **`-d 3`**: dimensionality. Use `3` for 3D volumes.
- **`-i` input**: any image. For labelmaps, pass the labelmap directly — don't binarize first unless you have a reason.
- **`-r` reference**: defines the output image grid. `antsApplyTransforms` uses only the geometry (size, spacing, direction, origin), not the pixel values. See the "reference image role" section below.
- **`-n` interpolation**: `Linear` for single-subject intensity, `BSpline[3]` for averaged templates, `GenericLabel` for labelmaps / masks (**not** `NearestNeighbor`). See [masks.md](masks.md).
- **`-u` output type**: force the output pixel type. Important for masks (`uchar`) to avoid float64 outputs with numerical jitter. Without `-u` the output dtype is typically `float64` even for integer inputs.
- **`-t` transform**: can be repeated. Applied in command-line order (leftmost first). Accepts `.mat`, `.h5`, `.nii.gz` (displacement field). See [transforms.md](transforms.md).

## `[path,1]` — invert an affine

Appending `,1` to a transform path tells `antsApplyTransforms` to invert it on the fly:

```
-t [reg_0GenericAffine.mat,1]   # use inverse of this affine
```

Works for affine transforms (`.mat`, `.h5`). **Does NOT work for displacement fields** (`.nii.gz`) — use the explicit pre-computed `reg_1InverseWarp.nii.gz` rather than trying `[reg_1Warp.nii.gz,1]`.

## Reference image role

The `-r` reference image defines the output grid only: its *size*, *spacing*, *direction cosines*, and *origin*. Pixel values are ignored. The transform chain is expected to map from a reference-space point to an input-space point (see [transforms.md](transforms.md) for why the chain works "backwards" from what you might think).

If you want a custom output grid — for example, a rotated frame aligned with the template axes but extended to contain a subject-specific region — build a reference image with the right geometry and save it to disk. Common pattern:

```python
import SimpleITK as sitk

ref = sitk.Image(size, sitk.sitkFloat32)        # size = (nx, ny, nz)
ref.SetSpacing(spacing)                         # e.g. (0.1, 0.1, 0.1)
ref.SetDirection(direction)                     # 9-tuple, same as template
ref.SetOrigin(origin)                           # world coordinate of voxel (0,0,0)
sitk.WriteImage(ref, "rotated_frame_reference.nii.gz")
```

To extend an existing grid (e.g. the template) to also contain another anatomy (e.g. a subject mask's bounding box rotated into the template frame): express both point sets in the template's orthonormal direction basis, take the axis-aligned union, then convert the resulting min/max back to a world origin and voxel size.

## `CreateJacobianDeterminantImage`

Compute the Jacobian determinant of a warp field at every voxel. Useful for diagnosing registration quality — extreme compression/expansion shows as large negative/positive log-Jacobian values.

```
CreateJacobianDeterminantImage DIM WARP OUT [doLog] [useGeometric]
```

Example:

```bash
CreateJacobianDeterminantImage 3 reg_1Warp.nii.gz jac_log.nii.gz 1 1
```

- **`doLog = 1`**: take the log of the determinant. Symmetric around 0 — negative = compression, positive = expansion. Almost always what you want.
- **`useGeometric = 1`**: use the geometric form. Physically meaningful and stable; set this.
- **Output space**: lives in the storage space of the warp (= fixed / template grid). To sample it in another space, resample through the appropriate transform chain.

## Displacement magnitude from a warp field

No ANTs binary for this; compute it directly in SimpleITK:

```python
import SimpleITK as sitk
import numpy as np

warp = sitk.ReadImage("reg_1Warp.nii.gz", sitk.sitkVectorFloat64)
arr  = sitk.GetArrayFromImage(warp)                    # shape (Z, Y, X, 3)
mag  = np.linalg.norm(arr, axis=-1).astype(np.float32)
mag_img = sitk.GetImageFromArray(mag)
mag_img.CopyInformation(sitk.VectorIndexSelectionCast(warp, 0))
sitk.WriteImage(mag_img, "disp_mag.nii.gz")
```

Large displacement = SyN needed to correct a lot at that voxel; correlates with regions where the registration was difficult. A companion field to the log-Jacobian — the Jacobian tells you *how* the warp distorted (compress/expand), the displacement magnitude tells you *how much* it moved things.

## `antsRegistration` quick notes

Full coverage of `antsRegistration` is out of scope here — the command has dozens of flags. The ones that commonly need attention:

- **`-x` masks**: per-stage `[fixed_mask,moving_mask]`. Passing a single global `-x` mask for all stages causes rigid/affine failures because the mask clips the image before global alignment. Use `NULL` masks for linear stages (full field of view), only bind a mask in the SyN stage where the metric needs a region of interest.
- **`-t` stage transforms**: `Rigid[step]`, `Affine[step]`, `SyN[step,update,total]`. The output `reg_0GenericAffine.mat` collapses the rigid + affine stages; there is no separate rigid file.
- **Metric mask vs soft mask**: if you want to softly down-weight skull in the registration metric, apply a soft (0–1 fractional) mask by **multiplying the moving image** before registration, not by passing it as the `-x` moving mask. The `-x` mask is a binary metric mask — fractional values underweight the boundary and confuse optimization.

## Common ANTs file outputs from SyN registration

After running `antsRegistration --transform SyN[...]`:

| file | contents | notes |
|---|---|---|
| `reg_0GenericAffine.mat` | composite affine (rigid + affine stages) | ITK transform file |
| `reg_1Warp.nii.gz` | forward displacement field | stored on fixed grid; displaces fixed → pre-affine-moving |
| `reg_1InverseWarp.nii.gz` | inverse displacement field | stored on fixed grid; displaces (post-inverse-affine) → fixed |
| `reg_Warped.nii.gz` | moving image resampled into fixed space | only written if `-o [prefix,WARPED,INVWARPED]` form used |
| `reg_InverseWarped.nii.gz` | fixed image resampled into moving space | same — requires the three-element `-o` form |

See [transforms.md](transforms.md) for how to chain these into forward/inverse transforms.
