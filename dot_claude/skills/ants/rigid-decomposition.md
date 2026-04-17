# Polar decomposition of `reg_0GenericAffine.mat`

How to extract the rigid (rotation + translation) component of an ANTs affine. Useful for building a "native-shape, template-axis-aligned" comparison frame where you want to rotate subject anatomy to match template axes without distorting it via scale/shear.

## Code

```python
import SimpleITK as sitk
import numpy as np

tx = sitk.ReadTransform("reg_0GenericAffine.mat")
params = np.asarray(tx.GetParameters())        # length 12: 9 matrix + 3 translation
fixed  = np.asarray(tx.GetFixedParameters())   # length 3: center of rotation

M = params[:9].reshape(3, 3)
t = params[9:12]
c = fixed[:3]

# Polar decomposition via SVD: M = U Σ V^T  →  R = U V^T
u, _, vt = np.linalg.svd(M)
R = u @ vt
if np.linalg.det(R) < 0:          # reflection — flip to stay in SO(3)
    u[:, -1] *= -1
    R = u @ vt

rigid = sitk.AffineTransform(3)
rigid.SetMatrix(R.flatten().tolist())
rigid.SetTranslation(t.tolist())
rigid.SetCenter(c.tolist())
sitk.WriteTransform(rigid, "rigid_only.mat")
```

## Why read parameters directly

Using `tx.GetParameters()` and `tx.GetFixedParameters()` sidesteps the `sitk.ReadTransform → sitk.AffineTransform` downcast, which can fail on some `.mat` files depending on how ITK wrote them. The generic `Transform` base class always exposes these.

## Properties of the result

- The rigid preserves the affine's center of rotation `c` and translation `t`. At `x = c`, `rigid(c) = c + t = affine(c)` — they agree exactly at the center.
- Away from the center, the rigid diverges from the full affine by exactly the scale/shear component. For typical brain registrations with singular values like `[1.05, 1.00, 0.98]`, the divergence at 1 cm from the center is sub-millimeter.
- The rotation is proper (`det(R) = +1`) by construction — the SVD-based decomposition with the reflection fix-up guarantees you stay in SO(3).

## Rotation angle

To report the rotation angle (useful for sanity-checking that the registration converged to something reasonable):

```python
angle_deg = np.degrees(np.arccos(np.clip((np.trace(R) - 1) / 2, -1, 1)))
```

Typical mouse head-in-coil rotations are 5–20°. Anything over 30° suggests a head positioning problem or a registration that fell into a wrong basin.

## Point direction of the extracted rigid

The rigid inherits the affine's point direction: `rigid.TransformPoint(p)` maps in the same direction as the parent affine, which in ANTs convention is **fixed → moving**. For mouse MRI (`fixed = template`, `moving = subject`), `rigid.TransformPoint(template_point) = subject_point`.

This means the rigid can be slotted into an `antsApplyTransforms` chain exactly where you'd put the affine in the same direction. For a `template → rotated-frame` resampling (rotated frame lives in template-oriented physical space), the rigid goes at the start of the chain (applied first, to take a rotated-frame point to a subject point — since the rotated frame is physically the same coordinate system as the template):

```
-t rigid_only.mat -t [reg_0GenericAffine.mat,1] -t reg_1InverseWarp.nii.gz
```

See [transforms.md](transforms.md) for the full explanation of why this chain works.
