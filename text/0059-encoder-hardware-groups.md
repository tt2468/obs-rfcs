# Summary

Implement a vendor-agnostic API allowing efficient handling of hardware device resources for encoders.

# Motivation

OBS's encoder stack is, to put it lightly, somewhat obtuse. Implementations of hardware encoders like NVENC which got by with gradual modifications to the (originally host-frame-only for things like x264) libobs API are now showing their age with the looming trend of clientside encodes (simulcasting).

There is a long list of drawbacks, both user-facing and backend-facing, to the existing encoder system. With a majority of OBS users now using hardware-accelerated encoders like QSV, NVENC, and AMF instead of the older software counterparts, OBS's current support framework for these encoders imposes many pipeline inefficiencies and heavily-duplicated code. Future popularity of simulcasting is expected to only worsen this problem.

Among the list of user-facing drawbacks are (with the solutions this RFC offers):
- Lack of support for reliable multi-device encoding
  - Vendor implementations of encoder accels are given the flexibility to use their vendor-specific APIs to perform peer-to-peer data transfer of frames provided from host memory or shared GPU textures
- Confusing encoder confuration UI (Previous example: `NVIDIA NVENC H.264 (New)`)
  - Encoders like NVENC AVC only require one implementation. Whether or not shared-texture support exists on a specific platform is irrelevant, as those details are dealt with by the encoder accel.
- Limited feature availability on specific platforms (Previous example: Newer NVENC features being implemented in jim-nvenc before its counterpart `ffmpeg_nvenc`)
  - As stated above, since encoders can now be implemented once, feature lag no longer a concern.

Among the list of backend-facing drawbacks are:
- Heavily duplicated code:
  - We have two NVENC implementations, one for shared-texture encoding and one for standard
  - Many places in the UI code currently have hardcoded option filters and fallbacks for encoders, which would largely be made unnecessary via this RFC
    - This RFC's changes would allow libobs to (optionally, but by default) automatically handle platform-specific limitations like lack of shared-texture encoding support, without spaghetti code in the UI
- On platforms which do not have reliable support for shared-texture encoding (Linux), encoding a video ladder requires downloading a texture for each ladder step to the host, then immediately uploading it to the encode device.
  - Shared-texture support can never be guaranteed, and the issue of missing texture-sharing support for encoders is unlikely to magically go away in the future.
    - Many vendors, though, do have some kind of scaling support on the hardware, allowing us to remove this performance loss as best as possible.
  - Some more exotic devices like the Xilinx MA35D transcoder card have almost unheard-of levels of encode performance, but will never be able to use true shared-texture encoding due to being descrete hardware. 

The theory of this RFC and the details of the implementation are inspired by FFmpeg's avutil `AVHWDeviceContext` API, which separates funamental video blocks like decoders, scalers, filters, and encoders away from the "glue" like device memory management and DMA.

References:
- `hwcontext.h`: https://ffmpeg.org/doxygen/trunk/hwcontext_8h_source.html
- FFmpeg HWAccelIntro wiki page: https://trac.ffmpeg.org/wiki/HWAccelIntro
- FFmpeg `hw_decode.c` example code: https://www.ffmpeg.org/doxygen/3.4/hw__decode_8c_source.html

# Implementation

The order of priorities for implementation are:
- Maintain feature parity with the existing libobs encoder framework.
- Maintain *to the best of our abilities* the same interface which the UI and third-party "stream management" plugins use to create and manage encoders.
  - Encoder accels should be able to be managed manually by the UI or a plugin, or in the case that an encoder is created which requires an accel to operate but is not assigned one, libobs will create and assign one automatically.
- Lay a path for the removal of the to-be-deprecated code paths in a major version release occuring at least one cycle after the introduction of this RFC.
- Lay the foundation blocks for the ability to accomplish the following things:
  - Usage of fixed-function scaling logic on encoder devices, which would otherwise go unused in lieu of consuming general-purpose GPU resources.
  - Multi-device encoding: Improving system encode performance with minimal possible system resources impact by leveraging device-side scaling features and peer-to-peer data transfer.
  - Possible efficiency improvements to other pipeline stages, with minimal code duplication, like the `ffmpeg_source` or WebRTC video conferencing features.

## New APIs

Note: As with any RFC, APIs which are described often are not mirrored identically when actually implemented. Methods of minor significance and info struct members have been intentionally left out.

The following APIs will be added:

### Enums:
- `obs_hardware_device_type`:
  - `OBS_HARDWARE_DEVICE_NONE`
  - `OBS_HARDWARE_DEVICE_CUDA`
- `obs_encoder_scale_type`
  - `OBS_ENCODER_SCALE_NONE`
  - `OBS_ENCODER_SCALE_SOFTWARE`
  - `OBS_ENCODER_SCALE_GPU`
  - `OBS_ENCODER_SCALE_HARDWARE_DEVICE`

### Structs:
- `struct obs_hardware_frame`
  - Struct containing metadata about a device-side surface/texture, contains opaque `void*` to carry implementation-specific data

### Types:
- `obs_hardware_device_t`:
  - Inspired by `AVHWDeviceContext`
  - Multiple device instances may be created, for example one per GPU, or two per GPU, for different purposes
  - Designed to implement device DMA, memory management
- `obs_encoder_accel_t`
  - Object responsible for accepting host-side `struct encoder_frame` or GPU texture `.encode_texture` callbacks, then delegating `struct obs_hardware_frame` objects via new `.encode_device_frame` callback method
  - Specifies via `obs_hardware_device_type` enum in `obs_encoder_accel_info` which hardware device backend is used.
  - Created manually by higher level UI/plugin logic, or created automatically by libobs on encoder startup if required and not applied
  - Can advertise support for the following capabilities:
    - `OBS_ENCODER_ACCEL_VIDEO` - Supports video
    - `OBS_ENCODER_ACCEL_DEVICE_SCALING` - Supports scaling of input frames for child encoders
    - `OBS_ENCODER_ACCEL_DYNAMIC_DEVICE_SCALING` - Supports modification of output ladder steps on-the-fly
    - `OBS_ENCODER_ACCEL_PEER_TO_PEER` - Supports passing textures internally between devices.
      - Likely to be largely unenforceable ("trust based") from a libobs perspective, as each vendor supports its own systems and techniques for p2p. Peer to peer routing likely to be configured via encoder group settings object.

### Methods:

Hardware Devices:
- `obs_hardware_device_t *obs_hardware_device_create(obs_hardware_device_type type, obs_data_t *settings)`
  - Create a hardware device, this is currently only meant to be called by `obs_encoder_accel_t` implementations
- `void obs_hardware_device_convert_encoder_frame(obs_hardware_device_t *device, struct encoder_frame *in, struct obs_hardware_frame *out, bool copy)`
  - Will defer to the GPU gods if this one makes sense, libavutil uses quite a different system (`av_hwframe_transfer_data()`)
- `void obs_hardware_device_destroy(obs_hardware_device_t *device)`
- (Likely others)

Encoder Accel Groups:
- `obs_encoder_accel_t *obs_encoder_accel_create(obs_hardware_device_type type, obs_data_t *settings)`
  - Creates a new encoder accel. Implementation creates hardware devices and stores internally as necessary. May pass along settings to created hardware devices.
  - Hardware device and encoder accel APIs are separated in order to accomodate future extensions to the hardware device usages in OBS.
  - Ties between hardware devices and accels are intentionally loose to allow maximum implementation flexibility
- `void obs_encoder_accel_update(obs_encoder_accel_t *accel, obs_data_t *settings)`
  - Updates settings object for accel.
- `void obs_encoder_accel_set_video(obs_encoder_accel_t *accel, video_t *video)`
- `bool obs_encoder_accel_enum_encoders(obs_encoder_accel_t *accel, size_t encoder_idx, obs_encoder_t **encoder)`
  - Used to enumerate child encoders of an accel
- `void obs_encocer_accel_output_frame(obs_encoder_accel_t *accel, size_t encoder_idx, struct obs_hardware_frame *out)`
  - Used by encoder accels to deliver hardware frame structs to encoders
- `void obs_encoder_accel_destroy(obs_encoder_accel_t *accel)`
- (Likely others)

Encoders:
- The `.encode_device_frame` callback will be added to `obs_encoder_info`
- `void obs_encoder_set_accel(obs_encoder_t *encoder, obs_encoder_accel_t *accel)`
- `obs_encoder_scale_type[] *obs_encoder_get_scale_types(obe_encoder_t *encoder)`
  - Retreive an array of supported encoder scale types, based on current configuration state
- The following existing encoder methods will gain implicit support for encoder accels:
  - `obs_encoder_set_frame_rate_divisor()` - If accel group is explicitly set and scaling is supported
  - `obs_encoder_get_frame_rate_divisor()` - If accel group is explicitly set and scaling is supported
  - `obs_encoder_set_scaled_size()` - If accel group is explicitly set and scaling is supported
  - `obs_encoder_get_width()`
  - `obs_encoder_get_height()`
  - `obs_encoder_get_video()`
  - (Possibly others)
- The following existing encoder methods will become no-op if an accel group is explicitly set:
  - `obs_encoder_set_video()`
  - `obs_encoder_set_preferred_video_format()`
  - (Possibly others)

## Deprecations:

- The `OBS_ENCODER_CAP_PASS_TEXTURE` cap will be deprecated in favor of encoder groups. All associated things will also be deprecated.

## Implementation process

The following steps will need to be taken to implement this RFC:

- Implementation of `obs_hardware_device_t`
  - Basic API support for passing GPU texture data or `struct encoder_frame` data as input, basic support for `struct obs_hardware_frame` output
- Implementation of `obs_encoder_accel_t` and encoder-side methods/functionality
  - Rerouting of `.encode` and `.encode_texture` callbacks to accel group if encoder has one
  - Lots of other things
- Gradual migration of existing hardware-accelerated encoder implementations to encoder accel system
  - NVENC gets a new encoder, accepting *only* hardware frames from a CUDA hardware device/accel. CUDA hardware device and accel implemented to accept encoder frames or GPU textures and convert/DMA them to CUDA textures.
  - Other encoders get similar migration
- Deprecations mentioned above

# Drawbacks

There are some drawbacks:
- This will take EFFORT, and will likely require multiple mentally dedicated developers
- There will be additional code burden
- There is a small risk that technologies like simulcast do not pan out, and this functionality could go largely unused

### Effort

Doing this kind of refactor is not simple, and is ultimately highly nuanced. The nuance is amplified given the goal of minimally impacting the existing encoder API. There are lots of nuggets of functionality glued to the encoder system which must in some way be preserved with this new system.

Writing the libobs APIs and integrating with the libobs code may end up being the most challenging part. It will require careful addition into the video pipelines of OBS. There are effectively two different pipelines, one for host-side frames and one for GPU textures, which must be modified to support rerouting of frame callbacks.

Luckily for us, I expect the migration of existing encoders to be less difficult, because:
- A notable portion of code should be able to be, to an extent, copy-pasted into the new descrete implementations of encoder/hardware devices
- Given the similarity of this API to FFmpeg's `AVHWDeviceContext` API, along with our existing reliance on FFmpeg libraries, much of the remaining functionality (host to device DMA, device memory management) can be almost-directly glued to libavutil. Device implementations can be migrated to native code at any later time.

### Code Burden

This adds a number of new types to libobs, which can be seen as not ideal in context of the codebase, but I believe the advantages of streamlining this system far outweigh the added code. Code which condensed many sequential operations like DMA and encode will now be logically separated into their own places, improving readability and understandability. Massive quantities of duplicated code will be able to be removed.

# Additional Information

None right now, phew.
