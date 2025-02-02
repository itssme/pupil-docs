---
permalink: /developer/core/overview
description: Welcome to Pupil Core developer documentation.
---

# Overview

Welcome to Pupil Core developer documentation. If you haven't already, we
highly recommend reading the [_Getting Started_](/core/#getting-started) section and the [_User Guide_](/core/software/pupil-capture.html), before continuing with the developer documentation.

Have a question? Get in touch with developers and other community members on the `#pupil-software-dev`
channel on [Discord](https://pupil-labs.com/chat).

<v-divider></v-divider>
## Where to start?

There are a number of ways you can interact with Pupil Core as a developer.
- **Use the realtime API**: The Pupil Core [Network API](/developer/core/network-api) allows you to remote control Pupil Core software and send/receive data realtime. It provides access to nearly all data generated by Pupil Core software: gaze, fixation, video data, and much more! The API can be used with any programming language that supports
[zeromq](https://zeromq.org/) and [msgpack](https://msgpack.org/).
- **Develop a plugin**: Pupil Core software is desgined with extendability in mind. It provides a simple yet powerful [Plugin API](/developer/core/plugin-api) that is used by nearly all existing Pupil Core components. Develop a plugin when you need to access to data that is _not provided_ via the Network API.
- **Modify source code**: Can't do what you need to do with the network based API or plugin? Then get ready to dive into the inner workings of Pupil, set up dependencies, and [run from source](#running-from-source)!

::: tip
<v-icon large color="info">info_outline</v-icon>
In most cases you can simply [download Pupil Core app bundles](https://github.com/pupil-labs/pupil/releases/latest) and extend the functionality via API or Plugin.
:::

## Terminology
There are a lot of new terms that are specific to eye and to Pupil Core. We have compiled a small list in the [terminology section](/core/terminology).


## Timing & Data Conventions
Pupil Capture is designed to work with multiple cameras that free-run at different
frame rates that may not be in sync. World and eye images are timestamped and any
resulting artifacts (detected pupil, markers, etc) inherit the source timestamp. Any
correlation of these data streams is the responsibility of the functional part that
needs the data to be correlated (e.g. calibration, visualization, analyses).

Example: Pupil Capture data format records the world video frames with their
respective timestamps. Independent of this, the recorder saves the detected gaze
and pupil positions at their own frame rate and with their timestamps. For details about
the stored data, see the [recording format](/core/software/recording-format "Pupil Core recording format") section.

Pupil Core software uses simple key-value structures to represent single data points.
Key-value structures can easily be serialized and nearly all programming languages have
an implementation for them.

Each data point should have at least two keys `topic` and `timestamp`. Each data point should be uniquely identifiable by its `topic` and `timestamp`.

1. `topic`: Identifies the type of the object. We recommend that you specify subtypes,
separated by a `.`
2. `timestamp`: [Pupil time](/core/terminology/#pupil-time) at which the datum was generated.

### Pupil Datum Format

The pupil detector generates `pupil` data from `eye` images. In addition to the `pupil`
topic and the `timestamp` (inherited from the eye image), the pupil detector adds fields most importantly:

- `norm_pos`: Pupil location in normalized eye coordinates, and
- `confidence`: Value indicating quality of the measurement

By default, the Pupil Core software uses the
[3d detector](/core/software/pupil-capture.html#pupil-detection) for pupil detection.
Since it is an extension of the 2d detector, its data contains keys that were
inherited from the 2d detection, as well as 3d detector specific keys. The minimal set of keys needed in a valid pupil datum object is: `id`, `topic`, `method`, `norm_pos`, `diameter`, `timestamp`, and `confidence`. Below you can
see the Python representation of a 3d pupil datum:

```python
{
    ### pupil datum required fields

    'id': 0,  # eye id, 0 or 1
    'topic': 'pupil.0',
    'method': '3d c++',
    'norm_pos': [0.5, 0.5],  # norm space, [0, 1]
    'diameter': 0.0,  # 2D image space, unit: pixel
    'timestamp': 535741.715303987,  # time, unit: seconds
    'confidence': 0.0,  # [0, 1]

    ### 2D model data

    # 2D ellipse of the pupil in image coordinates
    'ellipse': {  # image space, unit: pixel
        'angle': 90.0,  # unit: degrees
        'center': [320.0, 240.0],
        'axes': [0.0, 0.0],
    },

    ### 3D model data

    # Fixed to 1.0 in  pye3d v0.0.4.
    'model_confidence': 1.0,

    # pupil polar coordinates on 3D eye model. The model assumes a fixed
    # eye ball size. Therefore there is no `radius` key
    'theta': 0,
    'phi': 0,

    # 3D pupil ellipse
    'circle_3d': {  # 3D space, unit: mm
        'normal': [0.0, -0.0, 0.0],
        'radius': 0.0,
        'center': [0.0, -0.0, 0.0],
    },
    'diameter_3d': 0.0,  # 3D space, unit: mm

    # 3D eye ball sphere
    'sphere': {  # 3D space, unit: mm
        'radius': 0.0,
        'center': [0.0, -0.0, 0.0],
    },
    'projected_sphere': {  # image space, unit: pixel
        'angle': 90.0,
        'center': [0, 0],
        'axes': [0, 0],
    },
}

```

### Gaze Datum Format

Gaza data is based on one (monocular) or two (binocular) pupil positions. The gaze
mapper is automatically setup after calibration and maps pupil positions into the world
camera coordinate system. The pupil data on which the gaze datum is based on can be
accessed using the `base_data` key.

```python
{
    # monocular gaze datum
    'topic': 'gaze.3d.1.',
    'confidence': 1.0,  # [0, 1]
    'norm_pos': [x, y],  # norm space, [0, 1]
    'timestamp': ts,  # time, unit: seconds

    # 3D space, unit: mm
    'gaze_normal_3d': [x, y, z],
    'eye_center_3d': [x, y, z],
    'gaze_point_3d': [x, y, z],
    'base_data': [<pupil datum>]  # list of pupil data used to calculate gaze
}
```

```python
{
    # binocular gaze datum
    'topic': 'gaze.3d.01.',
    'confidence': 1.0,  # [0, 1]
    'norm_pos': [x, y],  # norm space, [0, 1]
    'timestamp': ts,  # time, unit: seconds

    # 3D space, unit: mm
    'gaze_normals_3d': {
        '0': [x, y, z],
        '1': [x, y, z],
    },
    'eye_centers_3d': {
        '0': [x, y, z],
        '1': [x, y, z],
    },
    'gaze_point_3d': [x, y, z],
    'base_data': [<pupil datum>]  # list of pupil data used to calculate gaze
}
```

### Surface Datum Format

Surface data is published when the [surface tracker](/core/software/pupil-capture/#surface-tracking)
is able to detect a defined surface. It includes
- the name of the detected surface,
- the timestamp of the video frame in which it was detected,
- the homographies to transform surface to image coordinates and vice versa,
- gaze and fixation data that was mapped onto the surface.

The gaze and fixation `norm_pos` fields contain [surface coordinates](/core/terminology/#surface-aoi-coordinate-system). The `base_data` field is a tuple of the
original topic and its timestamp.

```py
{
    "topic": "surfaces.surface_name",
    "name": "surface_name",
    "surf_to_img_trans": (
        (-394.2704714040225, 62.996680859974035, 833.0782341017057),
        (24.939461954010476, 264.1698344383364, 171.09768247735033),
        (-0.0031580300961504023, 0.07378146751738948, 1.0),
    ),
    "img_to_surf_trans": (
        (-0.002552357406770253, 1.5534025217146223e-05, 2.1236555655143734),
        (0.00025853538051076233, 0.003973842600569134, -0.8952954577358644),
        (-2.71355412859636e-05, -0.00029314688183396006, 1.0727627809231568),
    ),
    "gaze_on_surfaces": (
        {
            "topic": "gaze.3d.1._on_surface",
            "norm_pos": (-0.6709809899330139, 0.41052111983299255),
            "confidence": 0.5594810076623645,
            "on_surf": False,
            "base_data": ("gaze.3d.1.", 714040.132285),
            "timestamp": 714040.132285,
        },
        ...,
    ),
    # list of fixations associated with
    "fixations_on_surfaces": (
        {
            "topic": "fixations_on_surface",
            "norm_pos": (-0.9006409049034119, 0.7738968133926392),
            "confidence": 0.8663407531808505,
            "on_surf": False,
            "base_data": ("fixations", 714039.771958),
            "timestamp": 714039.771958,
            "id": 27,
            "duration": 306.62299995310605,  # in milliseconds
            "dispersion": 1.4730711610581475,  # in degrees
        },
        ...,
    ),
    # timestamp of the world video frame in which the surface was detected
    "timestamp": 714040.103912,
}
```

<v-divider></v-divider>

## Running From Source

Follow the setup instructions for your OS on the Pupil Core [Github repo](https://github.com/pupil-labs/pupil)

::: warning
<v-icon large color="warning">info_outline</v-icon>
When running from source, the [user settings](#installation) are not placed in the user's
home directory but in the root directory of the cloned repository.
:::
