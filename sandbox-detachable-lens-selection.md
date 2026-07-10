# Sandbox Detachable Lens Selection Interface

## Purpose

Describe the Sandbox lens focal length selection interface between camera (`BOSS`) and app (`WSDK`). With the introduction of user-provided detachable lenses, the user needs to curate a custom list of available lenses and quickly select the one currently in use.

## Camera UI Design

[Figma: SB Specifications](https://www.figma.com/design/7Ji5sTYtZwGbwrleclUR7P/SB-Specifications?node-id=32261-111976&t=6wPurxk8dpZAmSpy-4)

![Camera UI Design](https://raw.githubusercontent.com/tcamise-gpsw/redraft-test-repo/demo/lens-selection/images/image-20260624-000140.png)

## App UI Design

As of 7-1-2026, the app design is missing the curated list selection screen:

[Figma: KAT Quik App](https://www.figma.com/design/2DYSCNn09DslOsWHCK5Gu5/KAT-Quik-App?node-id=919-182897&m=dev)

![App UI Design](https://raw.githubusercontent.com/tcamise-gpsw/redraft-test-repo/demo/lens-selection/images/image-20260701-225505.png)

## Interface Requirements

- Retrieve the curated list of lenses (including the active lens) from the camera
- Be notified of changes to the curated list
- Set a lens from the curated list as active on the camera
- Get the complete list of supported lenses from the camera
- Add a lens from the supported list to the curated list
- Delete a lens from the curated list
- Know whether the camera supports the above operations

## FW Design

See: Sandbox FW - Lenses

---

## Proposal 1

The app discovers detachable lens support through new `settings.json` command IDs. Each operation maps to a BLE command with protobuf data.

### Command Specification

---

#### `WSDK_QUERY_ID_REQUEST_GET_DETACHABLE_LENS_SUPPORTED_LIST`

- **Params:** None
- **Response:** `WSDK_LensList`

Get the full list of lenses supported by the camera. The response contains two types of data: a range of mm describing the lens **parameters**, and a fixed list of specific **lenses**. Clients use this to create two selection mechanisms — numeric selection and fisheye lens selection. The `lenses` objects will only have `supported_lens_id` and `detachable_lens_name` fields populated.

---

#### `WSDK_QUERY_ID_REQUEST_GET_DETACHABLE_LENS_CURATED_LIST`

- **Params:** `WSDK_RequestCuratedLensList`
- **Response:** `WSDK_LensList`

Get the user's curated list of lenses for quick selection. The `register_for_updates` param specifies whether the client requests dynamic updates for active lens and curated list changes.

Returned `WSDK_LensList` objects include:
- `id` — lens identifier
- `focal_length` (mm) or `supported_lens_id` — one will be present
- `detachable_lens_name` — supplied when `supported_lens_id` is present
- `active` — `true` for the currently selected lens; missing or `false` otherwise

---

#### `WSDK_CMD_ID_REQUEST_ADD_DETACHABLE_LENS`

- **Params:** `WSDK_RequestAddUserLens`
- **Response:** `ResponseGeneric`

Add a lens to the curated list. Populate the request with either a `supported_lens_id` or a `custom_lens_focal_length` from the supported list response. Supplying both returns an error.

---

#### `WSDK_CMD_ID_REQUEST_DELETE_DETACHABLE_LENS`

- **Params:** `WSDK_RequestDeleteUserLens`
- **Response:** `ResponseGeneric`

Remove a lens from the curated list. Supply the `lens` object from the curated list query response.

---

#### `WSDK_CMD_ID_REQUEST_SET_ACTIVE_DETACHABLE_LENS`

- **Params:** `WSDK_RequestSetActiveLens`
- **Response:** `ResponseGeneric`

Set a lens as active on the camera. The lens must be from the curated list query response.

---

## Proposal 2

Proposal 1 leaks firmware implementation details into the app interface. Firmware stores catalog lenses as an enum and custom lenses as a float, serializes both directly, and makes the app figure out which one it's looking at.

The app only needs:

- An `id` to identify a lens
- A displayable `name` string
- A state flag (`isActive`)
- Optionally, a float `focalLength`

Everything else is firmware internals. If the camera resolves the factory/custom distinction at the boundary and always returns a uniform lens object, the app sees simple CRUD — with two Create methods since there are two distinct user inputs.

### Domain Model

One shared model for both factory and custom lenses containing all (and only) app-relevant fields.

### Operations

| Operation | Method | Description |
| --- | --- | --- |
| **Create (factory)** | Add from supported list | User picks from a chooser |
| **Create (custom)** | Add by focal length | User enters a number |
| **Read (supported)** | Get supported list | Full catalog of available lenses |
| **Read (curated)** | Get curated list | User's curated list with active state |
| **Update** | Set active lens | Select a lens from the curated list |
| **Delete** | Remove from curated list | Remove a lens from the user's list |

The two Create methods are not moving the firmware duality to a higher layer — they represent two real user actions: entering a number vs picking from a chooser.
