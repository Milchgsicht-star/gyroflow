<!-- cf04ec43-8a7b-4e59-a31a-3e59c6069b30 c18caa32-b35d-4f8e-8526-8df52648d98f -->
# Focal Length Stabilization

## Overview

Implement "zoom stabilization" that smooths out optical zoom changes by:

1. Plotting raw focal length data as a yellow curve in the timeline
2. Applying smoothing to the focal length curve
3. Using digital zoom to compensate for the difference between raw and smoothed focal length
4. Adding UI toggle and smoothing strength control

## Implementation Steps

### 1. Add focal length data extraction and storage

**File: `src/core/lib.rs` (StabilizationManager)**

- Add method to extract per-frame focal length data from `file_metadata.lens_params`
- Store both raw and smoothed focal length arrays similar to how `fovs` and `minimal_fovs` are stored
- Add fields to track focal length smoothing state

**File: `src/core/stabilization_params.rs` (StabilizationParams)**

- Add `focal_lengths: Vec<Option<f64>>` for raw focal length per frame
- Add `smoothed_focal_lengths: Vec<Option<f64>>` for smoothed values
- Add `focal_length_smoothing_enabled: bool` toggle
- Add `focal_length_smoothing_strength: f64` parameter (0.0 to 1.0)

### 2. Create focal length smoothing algorithm

**New file: `src/core/smoothing/focal_length.rs`**

- Implement a simple exponential smoothing or gaussian filter for scalar values (not quaternions)
- Take smoothing strength parameter (0.0 = no smoothing, 1.0 = maximum smoothing)
- Return smoothed focal length array
- Handle None values (frames without focal length data) gracefully

**File: `src/core/smoothing/mod.rs`**

- Export the new focal length smoothing module

### 3. Compute digital zoom compensation

**File: `src/core/lib.rs` or new `src/core/zooming/focal_length_compensation.rs`**

- Add method `compute_focal_length_zoom_compensation()`
- For each frame, calculate: `zoom_factor = smoothed_focal_length / raw_focal_length`
- This zoom factor should be multiplied with the existing FOV calculation
- Store compensation values that can be applied during frame transform

**File: `src/core/stabilization/frame_transform.rs`**

- Modify `FrameTransform::at_timestamp()` to apply focal length zoom compensation
- Integrate with existing zoom calculation (don't replace it, multiply/combine with it)

### 4. Add focal length visualization to timeline

**File: `src/ui/components/TimelineGyroChart.rs`**

- Add new chart data arrays: `focal_lengths: Vec<ChartData<1>>` and `smoothed_focal_lengths: Vec<ChartData<1>>`
- Extend `series` array from `[Series; 10]` to `[Series; 12]` (add 2 new series for focal length)
- Add method `setFromFocalLengthData()` to populate focal length data
- In `update_data()`, populate series[10] with raw focal length, series[11] with smoothed
- In `paint()`, add yellow color for raw focal length (#ffff00) and orange for smoothed (#ffaa00)
- Draw both curves when visible

**File: `src/ui/components/Timeline.qml`**

- Add UI buttons/checkboxes to toggle focal length curve visibility
- Add label "FL" (Focal Length) similar to Y/P/R labels
- Connect to chart visibility controls

### 5. Add controller methods

**File: `src/controller.rs`**

- Add Qt properties: `focal_length_smoothing_enabled: bool`, `focal_length_smoothing_strength: f64`
- Add methods to update timeline chart with focal length data
- Add methods to set/get smoothing parameters
- Trigger recompute when parameters change

### 6. Add UI controls

**File: `src/ui/menu/Stabilization.qml`** (or appropriate UI file)

- Add checkbox: "Stabilize focal length" to enable/disable feature
- Add slider: "Focal length smoothing" (0-100%) to control strength
- Add info tooltip explaining the feature
- Connect to controller properties
- Trigger recompute when changed

### 7. Integration and recomputation

**File: `src/core/lib.rs`**

- In `recompute_threaded()`, add step to compute focal length smoothing if enabled
- Extract focal length data from `lens_params` for all frames
- Apply smoothing algorithm
- Calculate zoom compensation factors
- Update FOV calculations to include focal length compensation
- Invalidate and recompute when focal length parameters change

### 8. Data flow updates

**File: `src/core/stabilization/compute_params.rs`**

- Add focal length related fields to `ComputeParams` if needed for passing data

**File: `src/core/zooming/mod.rs`**

- Update `get_checksum()` to include focal length smoothing parameters

## Key Technical Considerations

- **Coordinate with existing zoom**: Focal length compensation should multiply with existing adaptive zoom, not replace it
- **Handle missing data**: Some frames may not have focal length metadata
- **Normalize for display**: When plotting, normalize focal length to chart range (0-1)
- **Performance**: Smoothing should be efficient since it runs per frame
- **Edge cases**: Handle videos without focal length data gracefully (disable feature or show warning)

## Testing Approach

1. Load video with varying focal length (zoom lens footage)
2. Enable "No zooming" mode to see raw focal length display
3. Enable focal length stabilization
4. Verify yellow curve appears in timeline showing raw focal length
5. Adjust smoothing strength and verify smoothed curve updates
6. Verify digital zoom compensates for focal length changes
7. Test with existing stabilization modes (should work together)
8. Test with videos lacking focal length data (should gracefully disable/hide)

## Files to Modify/Create

**Core:**

- `src/core/lib.rs` - focal length extraction, smoothing trigger
- `src/core/stabilization_params.rs` - add parameters
- `src/core/stabilization/frame_transform.rs` - apply zoom compensation
- `src/core/stabilization/compute_params.rs` - pass focal length data
- `src/core/smoothing/focal_length.rs` - NEW: smoothing algorithm
- `src/core/smoothing/mod.rs` - export new module
- `src/core/zooming/mod.rs` - update checksum
- `src/core/zooming/focal_length_compensation.rs` - NEW (optional): zoom calc

**UI:**

- `src/ui/components/TimelineGyroChart.rs` - visualization
- `src/ui/components/Timeline.qml` - toggle buttons
- `src/ui/menu/Stabilization.qml` - controls
- `src/controller.rs` - Qt bindings

### To-dos

- [ ] Add get_focal_length_at_timestamp helper method to StabilizationManager in src/core/lib.rs
- [ ] Modify onProcessTexture callback in src/controller.rs to extract and display focal length when stabilization is off
- [ ] Test that focal length updates frame-by-frame with stabilization off and zoom stays at 100%