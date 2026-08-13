# OctoTuya protocol selection and connection detection

## Goal

Make OctoTuya work with Tuya devices using local protocol versions 3.1, 3.3, 3.4, or 3.5. Each configured plug chooses its own version. New plugs default to 3.3, and users can deliberately test or detect the correct version from the device editor.

The first deployment will be a clean uninstall/install. Existing `v33` settings do not need migration or compatibility handling.

## Scope

Change only protocol selection, connection diagnostics, related tests, and documentation. Preserve existing power control, status polling, G-code hooks, countdown timers, printer connection behavior, and system-command behavior.

The work remains in `amatsusah2/OctoTuya-SmartPlug`; no upstream pull request is required.

## Supported protocols

The allowed protocol values are:

- 3.1
- 3.3
- 3.4
- 3.5

The plugin must reject any other value and fall back safely to 3.3 for normal commands. A single backend constant is the source of truth for validation and detection order.

## Settings model

Replace the per-device `v33` Boolean with `protocolVersion`, stored as a string to match the HTML selector and avoid floating-point comparison concerns. New devices use `"3.3"`.

Before every normal TinyTuya command, the backend validates the configured value, converts it to a number, and calls `device.set_version(...)`. Protocol detection is never implicit in this path.

No settings-version bump or migration is required because deployment assumes a fresh installation.

## User interface

Replace **Use Protocol 3.3** with a per-device **Protocol version** dropdown containing the four allowed versions. The configured version is visible in the device summary.

Add a **Test & Detect** button to the device editor. It operates on the current form values, including unsaved IP address, device ID, local key, and selected protocol. While testing, the button is disabled and shows progress. The result is presented next to the control:

- Success: identify the working protocol and update the dropdown in the browser.
- Failure: report that no supported protocol returned valid status and suggest checking reachability, device ID, and local key.

Detection does not save settings automatically. The user must use OctoPrint's normal Save action.

## Diagnostic API

Add an administrator-only `testConnection` command to the existing Simple API. Its request contains the current form's IP address, device ID, local key, and selected protocol.

The backend validates required fields before connecting. It tries the selected version first, followed by the remaining supported versions in deterministic order. Each attempt:

1. Creates a fresh `tinytuya.OutletDevice`.
2. Sets the candidate protocol version.
3. Uses a bounded socket timeout.
4. Calls `status()` only; it never changes a DPS value.
5. Treats a dictionary containing a non-empty `dps` mapping as success.
6. Closes the device socket in a `finally` block.

Detection stops immediately on success. The response includes only success state, detected version, and sanitized error summaries. It never returns or logs the local key.

The diagnostic must not run during plugin startup, settings saves, polling, status checks, power commands, or retries of normal commands.

## Error handling and safety

- Missing or malformed fields return a clear validation error without opening a connection.
- Unsupported protocol values are not passed to TinyTuya.
- TinyTuya error dictionaries and exceptions are converted to short, credential-free summaries.
- Each candidate uses a new device object so a failed handshake cannot contaminate the next attempt.
- All sockets are closed, including after exceptions.
- The detector is read-only and issues only `status()`.
- Device credentials must not appear in committed tests, fixtures, documentation, or debug output.

## Files to change

- `octotuya_smartplug/__init__.py`
  - Add supported-protocol constants and validation helpers.
  - Replace `v33` defaults and hardcoded behavior with `protocolVersion`.
  - Add the administrator-only test/detect API path.
- `octotuya_smartplug/static/js/octotuya.js`
  - Add the 3.3 default for new devices.
  - Add Test & Detect request, progress, result, and dropdown-update behavior.
- `octotuya_smartplug/templates/octotuya_settings.jinja2`
  - Replace the checkbox with the dropdown.
  - Add the Test & Detect control and result area.
  - Display the chosen protocol in the plug summary.
- `tests/`
  - Add focused automated tests with mocked TinyTuya devices.
- `README.md` and `changelog.md`
  - Document supported versions, clean-install expectation, and Test & Detect behavior.

## Automated verification

Tests must cover:

- Allowed and rejected protocol values.
- Default protocol 3.3.
- Normal commands applying the configured protocol.
- Selected-version-first detection ordering.
- No duplicate candidate attempts.
- Immediate stop after a valid DPS response.
- Continued probing after TinyTuya error dictionaries and exceptions.
- All-versions-failed response.
- Socket closure on success and failure.
- Sanitized responses and logs that omit credentials.
- Administrator permission enforcement for the diagnostic API.

Run the full available test suite plus package/import checks before publishing.

## Device verification

After installing the fork cleanly in OctoPrint:

1. Re-enter the device settings.
2. Start with protocol 3.3 and run Test & Detect.
3. Confirm detection selects 3.4 for the target plug.
4. Save settings and restart OctoPrint.
5. Confirm status polling returns a valid DPS mapping without errors 904 or 914.
6. When safe for the printer, perform one deliberate on/off control test.

## Release and credential handling

Commit implementation and tests to the personal fork. Push using the user-authorized personal credential only through an ephemeral process environment. Do not save that credential in repository files, Git remotes, Git configuration, shell profiles, logs, or documentation.

## Decision rationale

- A reusable per-device selector was chosen over a one-line 3.4 patch because different devices may require different protocol versions.
- New devices default to 3.3, matching the plugin's previous explicit modern-protocol path.
- A clean install eliminates migration complexity and avoids the upstream migration function that resets the plug array.
- Test & Detect is explicit and user-triggered because failed protocol handshakes can be slow or briefly disrupt device connectivity.
- Detection updates the UI but does not save automatically, preserving OctoPrint's normal settings workflow.
