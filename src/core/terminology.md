---
permalink: /core/terminology
description: In this section, we will introduce a few key terms/concepts that are essential to Pupil Core.
---

# Terminology

## World
The term `world` generally refers to the subject's field of view. Sometimes it is used
in combination with other terms:

- **World Camera**: Physical scene camera that captures the subject's field of view.
- **World Process**: Application process responsible for processing the _world camera_
    video stream and [_gaze mapping_](#gaze-positions).
- **World Window**: The main Pupil Capture window; previews the result of the _world process_
- **World Coordinate System**: Refers to the [_Coordinate System_](#coordinate-system) of the _world camera_.

## Eye
The term `eye` is always used in combination with a different term that specifies the context:

- **Eye ID**: `0` or `1`; identifies the right and left eyes.
- **Eye Camera**: Physical camera capturing the subject's eye. There can be one or two
    eye cameras, depending on your setup.
- **Eye Process**: Application process responsible for processing the _eye camera_ video
    stream and [_pupil detection_](#pupil-positions).
- **Eye Window**: Previews the result of the _eye process_.
- **Eye Coordinate System**: Refers to the [_Coordinate System_](#coordinate-system) of the _eye camera_.
- **Eye Model**: Result of the _3d pupil detection_.

## Pupil Positions
Alternatively: _Pupil Data_. Output of the _pupil detection_. Location of the pupil
within the _eye coordinate system_.

- **Pupil Detection**: Process of detecting the pupil within the _eye camera_ video stream.
- **2d Pupil Detection**: Attempts to find a 2d ellipse that fits the pupil within a given _eye_ image.
    [Reference paper](https://arxiv.org/pdf/1405.0006.pdf).
- **3d Pupil Detection**: Pupil Core uses [`pye3d`](/developer/core/pye3d/) for 3d pupil detection.
    It implements a published mathematical 3D eye model capturing ocular kinematics and
    optics (see [Swirski and Dodgson, 2013](https://www.researchgate.net/publication/264658852_A_fully-automatic_temporal_approach_to_single_camera_glint-free_3D_eye_model_fitting "L. Świrski and N. A. Dodgson. A Fully-Automatic, Temporal Approach to Single Camera, Glint-Free 3D Eye Model Fitting. In Proceedings of ECEM 2013."),
    as well as [Dierkes, Kassner, and Bulling, 2019](https://www.researchgate.net/publication/333490770_A_fast_approach_to_refraction-aware_eye-model_fitting_and_gaze_prediction "K. Dierkes, M. Kassner, A. Bulling. A fast approach to refraction-aware eye-model fitting and gaze prediction. In ETRA ’19: Symposium on Eye Tracking Research and Applications, 2019.")).
    More specifically, frame-by-frame measurements of gaze direction and pupil size rely
    on eyeball-position estimates in the 3D coordinate system defined by the recording
    eye camera. [Read more about it here](/developer/core/pye3d).

## Gaze Positions
Alternatively: _Gaze Data_. Output of the _gaze estimation_. Location of the subject's
gaze within the _world coordinate system_.

- **Gaze Estimation**: Process of mapping a Pupil Position from the _eye coordinate system_
    to the _world coordinate system_.

## Confidence

Quality assessment of the _pupil detection_ for a given _eye_ image.
- `0.0` means that the pupil could not be detected.
- `1.0` highest possible value; the pupil was detected with very high certainty.

_Gaze data_ inherits the _confidence_ value from the _pupil data_ it was based on.

## Calibration

Process of understanding the relationship between _pupil positions_ and their
corresponding _gaze position_ within the _world coordinate stystem_.

During the _calibration_ process the subject is asked to fixate a list of target
(reference) locations within the _world camera_'s field of view. The Pupil Core software
collects these reference locations and _pupil data_ during the calibration. Afterwards,
it correlates them in time and calculates a mapping function that is used to estimate
the gaze for future pupil data.

## Timing

Pupil Core software uses timestamps with 64-bit floating point values. Their unit is
_seconds_. Pupil Core differentiates between to clocks:

- _System Time_: Returns devices current date time (not precise).
- _Pupil Time_: Synchronized clock used for time measurements (precise).

### Epoch

> The epoch is the point where the time starts [...]. For Unix,
the epoch is January 1, 1970, 00:00:00 (UTC).
- [https://docs.python.org/3/library/time.html](https://docs.python.org/3/library/time.html)

### System Time

The current date time of the device running the Pupil Core software. Uses the _Unix epoch_.
The _system time_ clock is not guaranteed to be monotonicly increasing since it is
subject to the devices network clock synchronization.

### Pupil Time

A precise monotonicly increasing clock for time measurements. Pupil Core software uses
it to timestamp all its generated data. This clock's _epoch_ is arbitrary. A fixed offset
can be applied between recordings to synchronize multiple devices running Pupil Core
software.

## Coordinate System

There are three coordinate systems for each camera:

- **2D Image Space**:
    - origin: top left
    - unit: pixels
    - includes lens distortion
    - bounds:
        - `x: [0, <image width>], y: [0, <image height>]`
    - example: `image shape: (800, 400), location: (400, 200)` (image center)
- **2D Normalized Space**:
    - origin: bottom left
    - unit: image width/height
    - includes lens distortion
    - equivalent to _2d image space_, normalized
    - bounds:
        - `x: [0, 1], y: [0, 1]`
    - example: `(0.5, 0.5)` (image center)
- **3D Camera Space**:
    - origin: center of the camera
    - does not include lens distortion
    - no boundaries
    - x: horizontal, y: vertical, z: optical axis
    - example:
        - `(0, 0, 1)` (a point on the optical axis)
    
[Reference](https://docs.opencv.org/2.4/modules/calib3d/doc/camera_calibration_and_3d_reconstruction.html).
You can use the _Camera Intrinsics_ to project a _3d camera location_ to _2d pixel location_, and vice versa.

### Surface (AOI) Coordinate System

[Surfaces](/core/software/pupil-capture/#surface-tracking) - also known as areas of interest or AOIs - define their own local coordinate system. For each scene image that includes sufficient surface markers, the `Surface Tracker` plugin calculates the respective _transformation_ function between the scene camera and surface coordinate system. The surface preview (red rectangle overlay) uses its inner triangle to indicate `up`/`top` within the local surface coordinate system. Pupil Capture and Player automatically map gaze and fixation data to surface coordinates if a valid surface transformation is available.

The **surface-normalized** coordinates of mapped gaze/fixations have the following properties:
- origin: bottom left (pay attention to red triangle indicating surface `up`/`top` side)
- unit: surface width/height
- bounds (if gaze/fixation is on AOI):
    - `x: [0, 1], y: [0, 1]`
- example: `(0.5, 0.5)` (surface center)

The lens distortion (camera intrinsics) are compensated for during the process of mapping data from the scene image space to the surface coordinate system. In other words, the surface coordinate system is not affected by scene camera lens distortion.


## Camera Intrinsics

The _camera intrinsics_ contain [camera matrix and lens distortion information](https://docs.opencv.org/2.4/modules/calib3d/doc/camera_calibration_and_3d_reconstruction.html).
They are used in _3d gaze mapping_ to correctly transform _3d pupil data_ to _3d gaze data_.

The Pupil Core software provides default camera intrinsics for all official Pupil Core
cameras. It is recommended to run the [Camera Intrinsics Estimation](/core/software/pupil-capture/#camera-intrinsics-estimation)
for your Pupil Core _world_ camera after receiving it. Each camera is slightly different
and running the estimation locally will result in slightly more precise gaze mapping.

## Fixations

Salvucci and Goldberg define different categories of fixation detectors. One of them describes dispersion-based algorithms:

> I-DT identifies fixations as groups of consecutive points within a particular dispersion, or maximum separation. Because fixations typically have a duration of at least 100 ms, dispersion-based identification techniques often incorporate a minimum duration threshold of 100-200 ms to help alleviate equipment variability.
>
> *--- Salvucci, D. D., & Goldberg, J. H. (2000, November). Identifying fixations and saccades in eye-tracking protocols. In Proceedings of the 2000 symposium on Eye tracking research & applications (pp. 71-78). ACM.*

Pupil Core's fixation detectors implement a dispersion-based method. The exact procedure differs depending on whether fixations are detected in an online or offline context:

### Online Fixation Detection

In Pupil Capture, fixations are detected based on a dispersion threshold in terms of degrees of visual angle with a minimum duration. Fixations are published as soon as they comply with the constraints (dispersion and duration). This might result in a series of overlapping fixations.

- **Maximum Dispersion (spatial, degree):** Maximum distance between all gaze locations during a fixation.
- **Minimum Duration (temporal, milliseconds):** The minimum duration in which the dispersion threshold must not be exceeded.
- **Confidence Threshold:** Data with a lower confidence than this threshold is not considered during fixation detection.

### Offline Fixation Detection

In Pupil Player, fixations are detected based on a dispersion threshold in terms of degrees of visual angle within a given duration window. The length of classified fixations are maximized within the duration window, e.g. instead of creating two consecutive fixations of length 300 ms it creates a single fixation with length 600 ms. Fixations do not overlap.

- **Maximum Dispersion (spatial, degree):** Maximum distance between all gaze locations during a fixation.
- **Minimum Duration (temporal, milliseconds):** The minimum duration in which the dispersion threshold must not be exceeded.
- **Maximum Duration (temporal, milliseconds):** The maximum duration in which the dispersion threshold must not be exceeded.
