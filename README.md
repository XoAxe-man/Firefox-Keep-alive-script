# Firefox Keep-Alive & Resource-Throttling Daemon

A lightweight, automated system designed to run a continuous, headless or low-resource Firefox instance controlled via Marionette. This system is engineered specifically for low-overhead environments, featuring aggressive memory management, automated page state restoration, and self-healing background execution.

## Architectural Overview

Instead of burdening lightweight network or storage nodes (like a NAS or basic ESXi hypervisor), this pipeline utilizes a dedicated Linux workstation during idle periods. 

To minimize bandwidth and compute overhead, the browser configuration enforces:
1. Strict memory and disk cache limits via a dynamic `user.js` profile.
2. Forced lowest-tier video quality selection and minimal reference frame rendering to conserve network pipes.
3. Automated tab recovery and interaction via direct JavaScript injection.

## Components

### 1. Profile Optimization (`user.js`)
On execution, the bash wrapper locates the active Firefox profile directory dynamically and drops a hardened `user.js` configuration file. This locks down internal parameters:
* Disables disk caching to eliminate I/O overhead.
* Caps memory cache tightly at 256MB.
* Restricts maximum DOM IPC process counts to prevent memory bloat over long deployment cycles.

### 2. Automation Control Loop (`firefox_refresh`)
A Python engine connects directly to the local browser instance via the Marionette driver on port 2828.
* **State Enforcement:** Injects JavaScript directly into the DOM browser storage (`localStorage`) to guarantee the stream initiates unmuted and at target default volume.
* **Playback Recovery:** Evaluates the state of HTML5 video elements via structural query selectors and triggers playback if a stalls/pauses are detected.
* **Session Teardown:** Implements strict exception handling inside a `finally` block to guarantee active Marionette sessions are systematically terminated, preventing socket leaks.

### 3. Production Service Deployment (`systemd`)
To ensure the pipeline operates as a true "fire-and-forget" infrastructure asset, execution is delegated to a local `systemd` user service daemon.

```ini
# ~/.config/systemd/user/firefox-marionette.service
[Unit]
Description=Firefox Marionette Background Service
After=network.target

[Service]
Environment="DISPLAY=:0"
ExecStart=/usr/bin/firefox --marionette
Restart=on-abnormal
RestartSec=5

[Install]
WantedBy=default.target
