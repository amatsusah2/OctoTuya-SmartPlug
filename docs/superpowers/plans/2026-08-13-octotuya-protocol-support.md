# OctoTuya Protocol Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-device Tuya protocol selection and an on-demand Test & Detect action, then publish an OctoPrint-installable GitHub ZIP URL.

**Architecture:** Keep the existing single-plugin structure. Centralize protocol validation and TinyTuya device construction in the Python backend, expose a read-only diagnostic through the existing Simple API, bind it to a small Knockout UI control, and retain all current on/off behavior.

**Tech Stack:** OctoPrint plugin API, Python, TinyTuya, Knockout.js, Jinja2, GitHub branch archives.

## Global Constraints

- Supported protocol values are exactly `3.1`, `3.3`, `3.4`, and `3.5`.
- New plugs default to `3.3`.
- Deployment is a fresh uninstall/install; do not implement `v33` migration or compatibility.
- Test & Detect is explicit and read-only; never invoke it during polling, startup, settings saves, status reads, or power commands.
- Do not add an automated test suite.
- Never log, commit, return, or persist the user's credential or a device local key outside normal OctoPrint settings.
- Live OctoPrint and device verification is performed manually by the user after push.
- The canonical installation URL is `https://github.com/amatsusah2/OctoTuya-SmartPlug/archive/refs/heads/main.zip`.

## File map

- `octotuya_smartplug/__init__.py`: protocol constants, validation, TinyTuya construction, detection logic, API response, fork update metadata, version.
- `octotuya_smartplug/static/js/octotuya.js`: new-device default, diagnostic request, progress/result observables, selected-version update.
- `octotuya_smartplug/templates/octotuya_settings.jinja2`: protocol dropdown, device-summary version, Test & Detect controls.
- `setup.py`: fork metadata and release version.
- `MANIFEST.in`: correct package paths so plugin assets are included in source distributions.
- `README.md`: supported protocols, detection workflow, and exact OctoPrint installation URL.
- `changelog.md`: 0.2.0 release notes.

---

### Task 1: Backend protocol selection

**Files:**
- Modify: `octotuya_smartplug/__init__.py`

**Interfaces:**
- Produces: `SUPPORTED_PROTOCOL_VERSIONS`, `DEFAULT_PROTOCOL_VERSION`, `normalize_protocol_version(value) -> str`, and `create_tuya_device(config, protocol_version=None)`.
- Consumes: existing plug dictionaries with `id`, `ip`, `localKey`, and `protocolVersion`.

- [ ] **Step 1: Add the protocol source of truth and helpers**

Add module constants after imports and methods near the utility section:

```python
SUPPORTED_PROTOCOL_VERSIONS = ("3.1", "3.3", "3.4", "3.5")
DEFAULT_PROTOCOL_VERSION = "3.3"


def normalize_protocol_version(value):
    value = str(value)
    return value if value in SUPPORTED_PROTOCOL_VERSIONS else DEFAULT_PROTOCOL_VERSION
```

Add a plugin helper:

```python
def create_tuya_device(self, config, protocol_version=None):
    version = normalize_protocol_version(
        protocol_version if protocol_version is not None else config.get("protocolVersion")
    )
    device = tinytuya.OutletDevice(config["id"], config["ip"], config["localKey"])
    device.set_version(float(version))
    return device
```

- [ ] **Step 2: Replace the settings default**

In `get_settings_defaults()`, replace:

```python
"v33": False,
```

with:

```python
"protocolVersion": DEFAULT_PROTOCOL_VERSION,
```

Do not change `get_settings_version()` or add migration code.

- [ ] **Step 3: Apply the selected version to every normal command**

Replace the direct `OutletDevice` construction and `v33` branch in `sendCommand()` with:

```python
device = self.create_tuya_device(plug)
```

Keep the existing command dispatch and return behavior unchanged.

- [ ] **Step 4: Run backend static checks**

Run:

```text
python -m py_compile octotuya_smartplug/__init__.py
rg -n "v33|set_version\(3\.3\)" octotuya_smartplug
```

Expected: compilation succeeds; the search returns no old `v33` or hardcoded 3.3 behavior.

- [ ] **Step 5: Commit backend protocol selection**

```text
git add octotuya_smartplug/__init__.py
git commit -m "feat: add per-device Tuya protocol selection"
```

---

### Task 2: Read-only Test & Detect backend

**Files:**
- Modify: `octotuya_smartplug/__init__.py`

**Interfaces:**
- Produces: `protocol_candidates(selected) -> list[str]`, `test_tuya_connection(config) -> dict`, and the `testConnection` Simple API command.
- Consumes: `create_tuya_device()` and the four unsaved form fields `ip`, `id`, `localKey`, and `protocolVersion`.

- [ ] **Step 1: Add deterministic candidate ordering**

```python
def protocol_candidates(selected):
    selected = normalize_protocol_version(selected)
    return [selected] + [
        version for version in SUPPORTED_PROTOCOL_VERSIONS if version != selected
    ]
```

- [ ] **Step 2: Add field validation and read-only detection**

Add a plugin method that requires non-empty `ip`, `id`, and `localKey`. For valid input, iterate over `protocol_candidates(config.get("protocolVersion"))`. Each attempt must create a fresh device, set a five-second socket timeout, call only `status()`, accept only a dictionary with a non-empty dictionary at `dps`, and close the socket in `finally`.

Use this return shape:

```python
{"success": True, "protocolVersion": version, "message": "Connected successfully."}
```

If every version fails, return:

```python
{
    "success": False,
    "message": "No supported protocol returned a valid status. Check the IP address, device ID, and local key.",
}
```

Do not include raw exceptions, payloads, or credentials in the response or logs.

- [ ] **Step 3: Expose the administrator-only API command**

Extend `get_api_commands()`:

```python
return dict(
    turnOn=["label"],
    turnOff=["label"],
    checkStatus=["label"],
    testConnection=["ip", "id", "localKey", "protocolVersion"],
)
```

Import `jsonify` inside the `testConnection` branch and return `jsonify(self.test_tuya_connection(data))`. Keep the existing `user_permission.can()` gate before all command dispatch.

- [ ] **Step 4: Run backend checks**

Run:

```text
python -m py_compile octotuya_smartplug/__init__.py
rg -n "localKey.*logger|logger.*localKey|except.*str\(" octotuya_smartplug/__init__.py
```

Expected: compilation succeeds; no diagnostic path logs credentials or raw exception strings.

- [ ] **Step 5: Commit detection backend**

```text
git add octotuya_smartplug/__init__.py
git commit -m "feat: detect Tuya protocol on demand"
```

---

### Task 3: Protocol selector and Test & Detect UI

**Files:**
- Modify: `octotuya_smartplug/static/js/octotuya.js`
- Modify: `octotuya_smartplug/templates/octotuya_settings.jinja2`

**Interfaces:**
- Consumes: `testConnection` API response with `success`, `protocolVersion`, and `message`.
- Produces: `protocolVersion` per plug plus root observables `testingConnection`, `connectionTestResult`, and `connectionTestSuccess`.

- [ ] **Step 1: Add diagnostic UI state**

In the view model constructor add:

```javascript
self.testingConnection = ko.observable(false);
self.connectionTestResult = ko.observable("");
self.connectionTestSuccess = ko.observable(false);
```

Reset the result when opening either `editPlug` or `addPlug`.

- [ ] **Step 2: Replace the new-plug default**

Replace:

```javascript
'v33':ko.observable(false),
```

with:

```javascript
'protocolVersion':ko.observable('3.3'),
```

- [ ] **Step 3: Add the diagnostic request**

Add `self.testConnection = function() { ... }` that reads `self.selectedPlug()`, validates the four observables exist, sets progress state, and POSTs:

```javascript
{
    command: "testConnection",
    ip: plug.ip(),
    id: plug.id(),
    localKey: plug.localKey(),
    protocolVersion: plug.protocolVersion()
}
```

On a successful API response, set `plug.protocolVersion(String(response.protocolVersion))`, mark success, and show `response.message`. On application failure or AJAX failure, show a credential-free error. Always clear `testingConnection` in `.always()`.

- [ ] **Step 4: Replace the settings controls**

In the summary row, replace the 3.3 check icon with text bound to `protocolVersion`.

In the editor, replace the checkbox with:

```html
<label class="control-label">{{ _('Protocol version') }}</label>
<select data-bind="value: protocolVersion" class="input-small">
    <option value="3.1">3.1</option>
    <option value="3.3">3.3</option>
    <option value="3.4">3.4</option>
    <option value="3.5">3.5</option>
</select>
```

Add a Test & Detect button bound to `click: $root.testConnection` and `disable: $root.testingConnection`, plus a nearby result span bound to the result and success state.

- [ ] **Step 5: Run frontend static checks**

Run:

```text
node --check octotuya_smartplug/static/js/octotuya.js
rg -n "v33|Use Protocol 3.3" octotuya_smartplug
```

Expected: JavaScript syntax succeeds; no old setting remains.

- [ ] **Step 6: Commit the UI**

```text
git add octotuya_smartplug/static/js/octotuya.js octotuya_smartplug/templates/octotuya_settings.jinja2
git commit -m "feat: add protocol detection controls"
```

---

### Task 4: Packaging, metadata, and documentation

**Files:**
- Modify: `setup.py`
- Modify: `octotuya_smartplug/__init__.py`
- Modify: `MANIFEST.in`
- Modify: `README.md`
- Modify: `changelog.md`

**Interfaces:**
- Produces: version `0.2.0`, fork URLs, corrected package manifest, and the documented install URL.

- [ ] **Step 1: Update version and fork metadata**

Set `plugin_version` in `setup.py` and `__plugin_version__` in `octotuya_smartplug/__init__.py` to `0.2.0`. Change `plugin_url` to the personal fork.

In `get_update_information()`, change the repository user to `amatsusah2` and the pip URL to the fork. Keep existing update-check behavior otherwise.

- [ ] **Step 2: Correct the source manifest**

Replace all `octoprint_tplinksmartplug` paths in `MANIFEST.in` with `octotuya_smartplug` so templates, translations, and static assets are packaged under the actual plugin name.

- [ ] **Step 3: Update user documentation**

In `README.md`:

- Replace the upstream install URL with `https://github.com/amatsusah2/OctoTuya-SmartPlug/archive/refs/heads/main.zip`.
- Document protocol 3.1/3.3/3.4/3.5 selection, the 3.3 default, and the on-demand Test & Detect action.
- State that settings must be re-entered after clean reinstall.

Add a `0.2.0` section to `changelog.md` describing the selector, detection action, and packaging correction.

- [ ] **Step 4: Verify package contents statically**

Run:

```text
python -m py_compile octotuya_smartplug/__init__.py
node --check octotuya_smartplug/static/js/octotuya.js
git diff --check
rg -n "andrelucca/OctoTuya-SmartPlug/archive/main.zip|octoprint_tplinksmartplug|v33" .
```

Expected: syntax checks pass; the search returns no stale install URL, wrong manifest package, or old setting.

- [ ] **Step 5: Commit release metadata and docs**

```text
git add setup.py octotuya_smartplug/__init__.py MANIFEST.in README.md changelog.md
git commit -m "docs: prepare OctoTuya 0.2.0"
```

---

### Task 5: Final verification, push, and install URL

**Files:**
- Verify all tracked changes; no new source files required.

**Interfaces:**
- Produces: pushed `main`, downloadable GitHub archive, and the user handoff URL.

- [ ] **Step 1: Run all agreed checks from a clean status view**

Run:

```text
python -m py_compile octotuya_smartplug/__init__.py
node --check octotuya_smartplug/static/js/octotuya.js
git diff --check origin/main...HEAD
git status --short
```

Expected: both syntax checks succeed, no whitespace errors, and only intentional commits are ahead of `origin/main`.

- [ ] **Step 2: Scan tracked content for credentials**

Search tracked files for personal-access-token prefixes and the known local key without printing matches. Abort the push if either is found.

- [ ] **Step 3: Push with ephemeral authentication**

Set the user-authorized token only in the current process environment, configure GitHub CLI's Git credential helper without persisting the token, push `main`, then immediately remove the environment variable. Do not rewrite the remote URL with embedded credentials.

- [ ] **Step 4: Validate the remote archive**

Download:

```text
https://github.com/amatsusah2/OctoTuya-SmartPlug/archive/refs/heads/main.zip
```

into a temporary directory. Confirm the ZIP opens and contains `setup.py`, `MANIFEST.in`, `octotuya_smartplug/__init__.py`, the JavaScript asset, and both Jinja templates.

- [ ] **Step 5: Handoff for manual verification**

Provide only the final installation URL prominently, followed by the short user-run checklist: clean uninstall/install, re-enter settings, run Test & Detect, confirm 3.4, save/restart, confirm status, and test on/off when safe.
