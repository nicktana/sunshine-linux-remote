# Linux remote desktop sessions {#linux_remote_sessions}

Design proposal based on checkout `6b176045`. The fork targets two workflows with a standard Moonlight client.

- **Physical desktop** attaches to an explicitly configured local seat. It must work at the display manager greeter after boot and recover across login, lock, logout, and display server replacement. The local and remote operators share the desktop.
- **Virtual desktop** provisions a separate graphical login under an authorized Linux account while the physical seat is still at the greeter or otherwise idle. It has its own display, input route, audio, and application lifecycle. Physical login and logout leave it running.

The initial compatibility target is one validated display manager and compositor combination for physical mode and one managed compositor for virtual mode. Both modes should be available on one host. The initial design uses separate host endpoints and limits each desktop to one controlling client. Supporting additional independent virtual desktops depends on resource limits and endpoint compatibility.

## Existing architecture

### Capture and encoding

[`src/platform/linux/misc.cpp`](../../src/platform/linux/misc.cpp) detects the window system from `WAYLAND_DISPLAY` and `DISPLAY` and probes capture sources compiled into the binary. Automatic probing considers NvFBC, wlroots, DRM/KMS, X11, KWin, and the XDG portal. X11 can also be probed as a fallback for software encoding with NvFBC. Explicit portal selection can fall back if the portal is unavailable. `platf::display_names()` and `platf::display()` enumerate outputs and open the selected backend for the requested encoder memory type.

| Backend | Source and prerequisites |
| --- | --- |
| X11 / NVIDIA NvFBC | [`x11grab.cpp`](../../src/platform/linux/x11grab.cpp) captures an existing X display. NvFBC provides NVIDIA capture for a compatible X11 environment. Both depend on an available X server and appropriate access. XWayland cannot expose the entire native Wayland desktop through X11 capture. |
| wlroots | [`wlgrab.cpp`](../../src/platform/linux/wlgrab.cpp) uses Wayland screencopy against compositor outputs. It requires the session socket and the supported protocol. |
| DRM/KMS | [`kmsgrab.cpp`](../../src/platform/linux/kmsgrab.cpp) reads DRM scanout planes and framebuffers. It requires an active captureable output and DRM access. Privileged framebuffer operations can require `CAP_SYS_ADMIN`. An explicitly configured KMS path can operate without a Wayland or X11 environment, subject to display and device access. |
| KWin | [`kwingrab.cpp`](../../src/platform/linux/kwingrab.cpp) requests an existing output through KWin's direct Wayland screencast protocol and consumes the associated PipeWire stream. Compositor support and authorization govern access. |
| XDG Desktop Portal | [`portalgrab.cpp`](../../src/platform/linux/portalgrab.cpp) negotiates RemoteDesktop plus ScreenCast or ScreenCast alone over the session bus and receives PipeWire frames. Consent and restore tokens are scoped to the portal backend and desktop. A saved token can become unusable. |

[`src/video.cpp`](../../src/video.cpp) probes encoder and display combinations and uses shared synchronous or asynchronous capture workers. Encoder compatibility depends on the capture buffer format, memory type, and GPU. Concurrent Moonlight streams and independent desktop targets are different levels of state. [`src/stream.cpp`](../../src/stream.cpp) has per-stream state alongside shared transport and broadcast workers.

### Input, audio, and application launch

- [`src/input.cpp`](../../src/input.cpp) decodes client input and dispatches through the platform implementation. Linux uses [the virtual HID adapter](../../src/platform/linux/input/virtualhid.cpp) and [shared virtual HID code](../../src/platform/virtualhid_input.cpp) for keyboard, pointer, and gamepad devices. Linux virtual device creation depends on uinput permissions. The shared code reads `XDG_SEAT` for seat assignment, which is useful for routing where honored. Capture permission and input permission are independent, including when the capture path negotiates a RemoteDesktop portal session.
- [`src/platform/linux/audio.cpp`](../../src/platform/linux/audio.cpp) connects to a PulseAudio-compatible server and captures from a sink monitor. The server, default sink, and monitor normally belong to the running user's audio environment. Each virtual desktop needs an explicit audio endpoint or a supported audio-disabled mode. Greeter audio is out of scope for the initial physical integration.
- [`src/nvhttp.cpp`](../../src/nvhttp.cpp) handles launch, resume, and cancel using the process-wide application manager `proc::proc` and hands launch metadata to [RTSP](../../src/rtsp.cpp). [`src/process.cpp`](../../src/process.cpp) manages the selected app, prep and undo commands, detached commands, the primary child, and process-group tracking. It augments its environment with client and stream metadata. On Linux, `platf::run_command()` starts children with the supplied environment and working directory as the calling identity. The Desktop entry can stream without launching a command. App launch has no independent login or account-switching step.

### Startup and session assumptions

The [packaged systemd user unit](../../packaging/linux/app-dev.lizardbyte.app.Sunshine.service.in) is wanted by `graphical-session.target`, orders after the graphical session, desktop autostart, and portal service, and delays launch briefly. It restarts on failure and exposes the `sunshine.service` alias. [CMake](../../cmake/prep/init.cmake) substitutes the native or Flatpak command. Desktop launchers and manual/SSH invocation are additional entry points. [Linux packaging](../../cmake/packaging/linux.cmake) also ships device permission rules.

The unit assumes a desktop and its environment already exist. A lingering user manager supplies a process supervisor. It still needs a compositor, display authorization, runtime directory, session bus and audio service where applicable. X11 may require `XAUTHORITY`. Wayland needs an accessible socket in the session runtime environment, and session services may require `DBUS_SESSION_BUS_ADDRESS`. A display manager can run the greeter under a separate identity and replace its display or revoke access when the seat changes owner.

### Upstream headless support

The upstream [Getting Started guide](https://github.com/LizardByte/Sunshine/blob/master/docs/getting_started.md) documents starting Sunshine over SSH with X11 and links to the [Remote SSH Headless Setup guide](https://docs.lizardbyte.dev/projects/sunshine/v0.22.1/about/guides/linux/headless_ssh.html). The latter demonstrates an externally started Xorg session with an NVIDIA virtual display and an explicit SSH bootstrap, without a local login, autologin, or dummy plug. This establishes that Sunshine can stream a separately provisioned headless graphical environment. It leaves boot-time provisioning, session ownership, login greeter integration, and connection routing to the host setup. The checkout's KWin capture opens an existing output, and its portal flow requests a source from an existing desktop. A virtual output inside a logged-in desktop still shares that desktop's login and compositor lifecycle.

## Proposed architecture

Use a local session manager to resolve each authorized launch to a stable desktop ID and a current backend generation. Start with a Sunshine worker process per desktop so the application manager, encoder probe state, capture workers, and platform environment remain process-local. Reuse Sunshine's pairing, discovery, RTSP, encoding, and input decoding. Start with distinct host endpoints and per-endpoint state directories, pairing identities, and port allocations. Validate all TCP and UDP listeners, discovery advertisements, and ordinary Moonlight pairing, launch, resume, and cancel before committing to that topology. Introduce a shared network entry point only if endpoint validation shows it is needed.

| Interface | Contract |
| --- | --- |
| Session manager | Resolve a configured desktop to its authorized client, mode, owner and seat policy. Acquire or resume a generation-bound lease, report readiness, and release or stop it. Serialize provisioning per desktop and bound startup, idle, and stop times. |
| Desktop backend | `prepare`, `watch`, and `stop` operations return or retire a desktop context. The physical backend observes the active logind session for the configured seat and attaches via the supported display-manager integration. The virtual backend provisions an administrator-configured compositor and desktop under the assigned account. |
| Capture backend | Probe capabilities and enumerate outputs within the desktop context. Open a specific output for an encoder memory type, and report source loss, permission failure, or format changes. Adapt existing `platf::display_t` implementations inside the worker. |
| Input backend | Open an injection route scoped to the same desktop generation and report supported device classes. Forward events only on a current control lease. Release keys, buttons and gamepads when the lease ends or the target changes. |
| Application and audio adapters | Launch allowed applications as the context owner using a constructed environment. Capture only the context's audio source. Track and stop session-owned children, including detached descendants, through a session scope where supported. |

A desktop context contains the stable ID, generation, mode, owner UID, optional logind session and seat identity, validated display endpoint, capture output identity, and input and audio endpoints. Provide the worker an allowlisted environment at process creation. Keep open resources tied to the generation and close them when it ends. Lifecycle states are stopped, starting, ready, draining, and failed. Acquisition succeeds only after capture is ready and, for a controlling lease, input is ready. Failed starts release processes, devices, and endpoint allocations before retry.

For physical mode, first validate active-session tracking and capture at the greeter on one display manager and GPU. DRM scanout is a candidate for capture across greeter and user transitions if it reliably identifies the configured output. A supported compositor or display-manager handoff can supply a session-specific capture path. On seat change, gate input and frame delivery, revoke the old generation, discard queued frames and events, and check the new target and both routes before reopening. The initial client experience can require reconnect after a transition. Preserve display manager login and lock enforcement. Offer a single desktop entry on the physical endpoint initially and disable app and prep commands there.

For virtual mode, start one dedicated compositor with a headless output, a session bus, and an optional audio service for the assigned account. Prototype with a wlroots compositor that exposes screencopy and a compositor-scoped input protocol. Check that the chosen versions expose both and can enforce local socket access. Keep a software rendering and encoding baseline to separate session lifecycle failures from GPU interoperability. Scope the first implementation to one virtual desktop per dedicated account. A per-account runtime directory, bus, and audio service can be shared by multiple sessions under the same UID, so multiple desktops per UID need separate routing and an explicit threat model. A virtual lease can survive disconnect for a configured idle interval, after which the manager drains the worker and reaps the desktop scope.

## Security boundaries

- Pair clients through the existing Sunshine mechanism, then authorize access by verified client certificate and configured desktop. Enable physical greeter access only through explicit host policy. Apply the same check to launch, resume, cancel, and control channel attachment. Require encrypted remote input and media for these endpoints. OS credentials entered at the greeter remain between the operator and display manager.
- Keep network listeners, codecs, pairing state, app commands and web administration in unprivileged processes. If physical capture or input needs privileged mediation, use a small local helper with a fixed seat/output policy, peer-credential checks, and revocable resources. Scope any privileged framebuffer access to that helper. A DRM descriptor can expose more than one output and may remain usable after handoff, so validate the lifetime and authority of anything exported. Prefer delivering frames over exporting unrestricted DRM access.
- Bind physical input to the approved seat through an integration that demonstrably respects routing. Give the virtual compositor its own input path and exclude host-wide uinput fallback. A seat environment variable alone does not enforce isolation. Require frame and input isolation tests before enabling either endpoint on a production host.
- Use administrator-owned account mappings and launch profiles. Use protected session directories, dedicated UIDs for isolated virtual desktops, and explicit rendering and device access. Do not expose arbitrary command, environment, device, UID, or path selection over the manager protocol. Keep pairing keys, portal restore tokens, Xauthority credentials, and greeter input out of logs. Audit authorization outcomes, lease transitions, and backend failures without recording input or frames.

## Implementation phases

1. Establish a disposable Linux VM fixture with a real display manager and a standard Moonlight client. Record the boot-to-greeter state, local input and capture behavior, current user-service behavior, listener layout, and the baseline GPU and software encoder results.
2. Add the desktop context and manager lifecycle with fake backends. Validate separate endpoint identity, concurrent routing, and cleanup while preserving the existing single-user path. Keep the new modes opt-in.
3. Implement virtual provisioning and isolated capture, input, and app launch. Verify boot-time readiness without a physical login. Add audio and hardware acceleration after the software path passes lifecycle and isolation tests.
4. Add the physical backend for the selected display manager and GPU. Validate greeter operation and transitions first in the VM and then on real hardware. Introduce the smallest necessary local helper if the validated path requires one.
5. Harden authorization and recovery, then document supported combinations and deployment. Add further compositor, display-manager, and GPU coverage only after recording the constraints of each combination.

## Testing strategy

- Add GoogleTest coverage using `tests/tests_common.h` for target resolution, authorization, capability negotiation, lease concurrency, generation invalidation, stale events, timeout and rollback, and child cleanup. Use fake backends and deterministic clocks. Target full coverage of changed code.
- In disposable VMs, boot with no physical user logged in. Pair, capture and control the greeter, log in, lock, log out, and restart the display manager. Run a virtual desktop concurrently and verify with recognizable frames, input events, and audio that neither endpoint crosses into the other. Repeat with disconnect, resume, and idle expiry.
- Inject failures during provisioning and handoff, including compositor crashes, worker crashes, unavailable audio, occupied ports, revoked device access, and encoder failures. Verify key release, removal of stale sockets and child processes, and isolation of cancellation and restart. Exercise unpaired, revoked, and cross-endpoint clients.
- Repeat relevant cases on supported GPUs and display-manager/compositor combinations. Include connector unplug, VT switch, GPU reset where reproducible, software versus hardware encoding, and current desktop user-service behavior. Use test-only configurations and private ports in disposable environments.

## Technical risks and acceptance gates

| Risk | Gate before enabling the affected mode |
| --- | --- |
| Greeter protocols, DRM ownership, and output identity differ by display manager, compositor, driver, and VT. Direct scanout, protected buffers, or a disconnected output may prevent capture. | Validate a named stack on hardware across boot, login, lock, logout, and VT switch. Treat unknown targets and invalid frames as unavailable. |
| Frame queues and injected key state can outlive a seat switch. Kernel virtual devices can reach unintended consumers. | Demonstrate generation-safe handoff, release of held input, and concurrent isolation. |
| Multiple Sunshine endpoints may collide on listeners, mDNS identity, pairing data, or Moonlight's host identity. | Confirm ordinary client discovery or manual add, distinct pairing, and concurrent streams across the entire port range. |
| Headless GPU allocation, DMA-BUF modifiers, and encoder limits vary. | Start with a software baseline, document format fallbacks, and enforce resource quotas and encoder admission limits. |
| Privileged capture widens the impact of a compromised worker or interface. | Verify helper identity and policy checks, descriptor lifetimes, revocation, and the absence of host privilege in network-facing processes. |
| Session and audio services can inadvertently bind to the console or another desktop under the same UID. | Validate per-desktop socket, bus, input, and audio selection with concurrent workloads and restart tests. |
