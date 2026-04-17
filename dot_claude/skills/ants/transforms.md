# Transforms, chains, and warp field semantics

Everything you need to reason correctly about composing ANTs transforms.

## Point direction vs image direction

**The single most important thing to remember:** ANTs transforms act on **points** in the direction `fixed → moving`, which is the **opposite** of the direction in which they "move images".

- Forward image warp: *moving image* resampled into *fixed space*.
- Forward point transform: *fixed point* mapped to *moving point*.
- Why: to resample the moving image onto the fixed grid, you iterate over fixed voxels and, for each one, ask *"what moving-space point does this correspond to?"* That's the fixed → moving point direction.

`fixed = reference = where output lives`, `moving = input = what gets warped`. In mouse MRI template building, `fixed = template`, `moving = subject`, so the forward warp goes *template → subject* in point direction and *subject → template* in image direction.

To verify empirically on unfamiliar data:

```python
import SimpleITK as sitk
tx = sitk.ReadTransform("reg_0GenericAffine.mat")
print(tx.TransformPoint((0, 0, 0)))  # expect a subject-like coordinate, since fixed=template
```

## `antsApplyTransforms` applies transforms left-to-right

**Leftmost `-t` is applied first.** This is the opposite of what the `antsApplyTransforms --help` text seems to describe (it talks about a stack where "each point is warped first according to the last transform pushed onto the stack", reading as right-to-left). **Ignore the help text.** The actual behavior matches the ANTs wiki prose and all working scripts in the wild.

```
antsApplyTransforms -t A -t B -t C
# application order: A first, then B, then C
```

Empirical proof: try `-t [affine,1] -t invwarp` and `-t invwarp -t [affine,1]` on the same template→subject resampling task. The first produces a meaningful inverse chain (affine brings the subject point into the template extent, then the inverse warp refines). The second silently degrades to affine-only because the subject point lands outside the inverse warp's extent. Only the first matches the registered transform.

## Warp field storage and semantics

The key to understanding ANTs warp fields:

- `reg_1Warp.nii.gz` (forward warp) is stored on the **fixed image grid**. Each voxel holds a displacement vector pointing **from fixed space toward "pre-affine moving space"** — i.e., the warp field does *not* map directly to moving space. The affine completes the mapping.
- `reg_1InverseWarp.nii.gz` (inverse warp) is also stored on the **fixed grid**. Each voxel displaces toward fully fixed space, **after the inverse affine has been applied**.

This is why the composition order matters:

| direction | meaning | application order | command line |
|---|---|---|---|
| **forward** (fixed → moving for points, = resample moving → fixed image) | warp first, then affine | `warp`, then `affine` | `-t reg_1Warp.nii.gz -t reg_0GenericAffine.mat` |
| **inverse** (moving → fixed for points, = resample fixed → moving image) | `[affine,1]` first, then inverse warp | `[affine,1]`, then `invwarp` | `-t [reg_0GenericAffine.mat,1] -t reg_1InverseWarp.nii.gz` |

Intuition for the forward chain: start with a fixed (template) point, apply the warp to get a pre-affine-moving point (still in fixed-grid physical space, since the warp field lives there), then apply the affine to land in true moving (subject) space. Intuition for the inverse: start with a moving (subject) point, apply the inverse affine to land in pre-warp template space, then apply the inverse warp to refine to the true template point.

## Warp field extent trap

Displacement fields are only defined inside the voxel grid they were stored on. **Outside that extent, the displacement silently extrapolates to zero**, meaning the warp "does nothing" for those points. The chain then degrades to the affine-only mapping and the output looks *mostly* right but is subtly off.

This bites most often when you're composing an inverse chain and the subject physical extent doesn't overlap the template physical extent (e.g., mouse MRI where the subject z range is 5–18 mm and the template z range is −7 to +1 mm). If you feed the warp a subject-space point directly, you get zero displacement because the point is outside the template grid.

**Symptom**: `-t [affine,1] -t invwarp` and `-t [affine,1]` alone produce identical results. Means the warp isn't contributing — the subject point never reaches the warp's extent.

**Fix**: make sure the chain brings the point *into* the warp's stored extent *before* the warp is applied. For the forward chain this is automatic (start with a template point, apply warp first — the point is in the template grid from the start). For the inverse chain, the `[affine,1]` step is what moves the point from the subject's physical extent into the template's physical extent; only then can the inverse warp meaningfully refine it.

## Debugging tip: two-step resampling

When a single composed chain produces a misaligned result, decompose it into two separate `antsApplyTransforms` calls where each step is a known-good chain:

```bash
# Step 1: template → subject (standard inverse chain, known to work)
antsApplyTransforms -i template_mask.nii.gz -r subject_ref.nii.gz \
  -o tmp_subject_space.nii.gz -n GenericLabel \
  -t "[reg_0GenericAffine.mat,1]" -t reg_1InverseWarp.nii.gz

# Step 2: subject → rotated (rigid only, trivially correct)
antsApplyTransforms -i tmp_subject_space.nii.gz -r rotated_ref.nii.gz \
  -o final.nii.gz -n GenericLabel -t rigid_only.mat
```

Compare `final.nii.gz` to whatever your single-call composed chain produced. If they differ, the single-call chain has an order or direction mistake. If they match, the alignment is correct and the bug is elsewhere (probably the reference grid, interpolation, or dtype).

## Adding a rigid rotation to a chain

If you're resampling into a custom frame defined by a rigid rotation of subject space (e.g. to align anatomical axes with image axes), the rigid transform is a point transform in the fixed → moving direction of the original affine — so for the template → rotated-frame chain, the rigid goes at the **start** (applied first, since you're starting from a rotated-frame point):

```
-t rigid_only.mat -t [reg_0GenericAffine.mat,1] -t reg_1InverseWarp.nii.gz
```

Applied in order: `rigid` takes a rotated-frame point to a subject point, `[affine,1]` takes that to a pre-warp template point, `invwarp` refines it to a true template point.

## Summary

- Transforms compose **left to right** on the `antsApplyTransforms` command line.
- Point direction is **fixed → moving**; image resampling is the opposite.
- Warp fields live on the **fixed grid**; outside that extent they silently no-op.
- Forward chain: `-t warp -t affine`. Inverse chain: `-t [affine,1] -t invwarp`.
- When in doubt, decompose into two-step resampling to isolate the bug.
