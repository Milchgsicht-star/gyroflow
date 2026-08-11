<!-- bf509d80-3293-4109-b94d-9f4e7abf6a95 08c96c7c-6d73-4ffe-acd2-be76f5136e5d -->
# Enable Focal Length Display Without Stabilization

## Overview

Currently, `current_focal_length` is only calculated when frames are actively processed for stabilization. When `stab_enabled` is false, the processing returns early and never extracts the focal length from metadata. This plan adds a lightweight path to calculate focal length using the same pipeline but without pixel processing.

## Implementation Steps

### 1. Add method to extract frame info without pixel processing

**File: `src/core/lib.rs`**

Add a new public method after `process_pixels` (around line 850):

```rust
pub fn get_frame_info(&self, mut timestamp_us: i64, frame: Option<usize>) -> Result<stabilization::ProcessedInfo, GyroflowCoreError> {
    let (offset, fps) = {
        let params = self.params.read();
        (params.frame_offset, params.fps)
    };
    let frame = frame.map(|x| (x as i32 + offset).max(0) as usize);
    timestamp_us += (offset as f64 / fps * 1000000.0).round() as i64;

    if let Some(scale) = self.params.read().fps_scale {
        timestamp_us = (timestamp_us as f64 / scale).round() as i64;
    }

    // Ensure everything is computed
    if self.smoothing_invalidated.load(SeqCst) {
        self.recompute_smoothness();
        self.smoothing_invalidated.store(false, SeqCst);
    }
    if self.zooming_invalidated.load(SeqCst) {
        self.recompute_adaptive_zoom();
        self.zooming_invalidated.store(false, SeqCst);
    }
    if self.undistortion_invalidated.load(SeqCst) {
        self.recompute_undistortion();
        self.undistortion_invalidated.store(false, SeqCst);
    }

    self.stabilization.read().get_frame_info_only(timestamp_us, frame)
}
```

### 2. Add method in Stabilization to get info without processing

**File: `src/core/stabilization/mod.rs`**

Add a new public method after the existing `process_pixels` method (around line 650):

```rust
pub fn get_frame_info_only(&self, timestamp_us: i64, frame: Option<usize>) -> Result<ProcessedInfo, GyroflowCoreError> {
    let frame = frame.unwrap_or_else(|| crate::frame_at_timestamp(timestamp_us as f64 / 1000.0, self.compute_params.get_scaled_fps()) as usize);
    
    let itm = if self.cache_frame_transform {
        self.stab_data.get(&timestamp_us)
    } else {
        Some(FrameTransform::at_timestamp(&self.compute_params, timestamp_us as f64 / 1000.0, frame))
    };

    if let Some(itm) = itm {
        Ok(ProcessedInfo {
            fov: itm.fov,
            minimal_fov: itm.minimal_fov,
            focal_length: itm.focal_length,
            backend: "Info only"
        })
    } else {
        Err(GyroflowCoreError::FrameTransformNotFound)
    }
}
```

### 3. Modify controller to use new method when stabilization is disabled

**File: `src/controller.rs`**

Modify the `onProcessTexture` callback (around line 998) to extract info even when disabled:

```rust
// Around line 998, replace:
if !stab.params.read().stab_enabled { return true; }

// With:
if !stab.params.read().stab_enabled {
    // Still calculate focal length info without processing
    let (offset, fps) = {
        let params = stab.params.read();
        (params.frame_offset, params.fps)
    };
    let frame_adjusted = (frame as i32 + offset).max(0) as u32;
    let timestamp_ms_adjusted = timestamp_ms + (offset as f64 / fps * 1000.0).round();
    
    if let Ok(info) = stab.get_frame_info((timestamp_ms_adjusted * 1000.0).round() as i64, Some(frame_adjusted as usize)) {
        update_info2((info.fov, info.minimal_fov, info.focal_length, QString::from("Stabilization disabled")));
    } else {
        update_info2((1.0, 1.0, None, QString::from("---")));
    }
    return true;
}
```

Modify the `onProcessPixels` callback similarly (around line 1131):

```rust
// Around line 1131, replace:
if !params.stab_enabled { return (0, 0, 0, std::ptr::null_mut()); }

// With:
if !params.stab_enabled {
    // Still calculate focal length info without processing
    let (frame_offset, fps_val) = (params.frame_offset, params.fps);
    drop(params);
    
    let frame_adjusted = (frame as i32 + frame_offset).max(0) as u32;
    let timestamp_ms_adjusted = timestamp_ms + (frame_offset as f64 / fps_val * 1000.0).round();
    
    if let Ok(info) = stab.get_frame_info((timestamp_ms_adjusted * 1000.0).round() as i64, Some(frame_adjusted as usize)) {
        update_info2((info.fov, info.minimal_fov, info.focal_length, QString::from("Stabilization disabled")));
    } else {
        update_info2((1.0, 1.0, None, QString::from("---")));
    }
    return (0, 0, 0, std::ptr::null_mut());
}
```

## Key Points

- Uses the exact same `FrameTransform` calculation pipeline as when stabilization is enabled
- The `FrameTransform::at_timestamp` method calls `get_lens_data_at_timestamp` which extracts focal length from `file_metadata.lens_params`
- No pixel processing overhead when stabilization is disabled
- The displayed focal length will be the raw value from metadata (not adjusted for zoom, since zooming isn't applied when disabled)
- Both GPU texture and CPU pixel processing paths are handled consistently

## Testing

After implementation:

1. Load a video with variable focal length (e.g., from Sony cameras with lens metadata)
2. Toggle stabilization on/off using the stabilization checkbox
3. Verify that the focal length line in the timeline and the text display update correctly in both states
4. Confirm the values match between enabled/disabled states (accounting for zoom when enabled)