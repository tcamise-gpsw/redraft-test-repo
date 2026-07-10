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

See: [Sandbox FW - Lenses](https://goproinc.atlassian.net/wiki/spaces/FEH/pages/1908768811)

---

## Proposal 1

The app discovers detachable lens support through new `settings.json` command IDs:

- `GPCAMERA_DETACHABLE_LENS_CURATED_LIST_GET`
- `GPCAMERA_DETACHABLE_LENS_ACTIVE_SET`
- `GPCAMERA_DETACHABLE_LENS_SUPPORTED_LIST_GET`
- `GPCAMERA_DETACHABLE_LENS_ADD_LENS`
- `GPCAMERA_DETACHABLE_LENS_DELETE_LENS`

Each maps to a BLE command with protobuf data support.

### Protobufs

```protobuf
/**
 * An individual lens object
 *
 * @external
 */
message WSDK_Lens {
    optional int32       lens_id = 1;  // Lens ID, to be used for api operations, should be treated as ephemeral.
    optional float  focal_length = 2;  // Focal Length of the lens in mm
    optional string    lens_name = 3;  // The string representation of the supported lens
    optional bool         active = 4;  // When set to true, is the active lens on the camera. Output only.
}

message WSDK_LensParameters {
    optional float minimum_focal_length = 1;  // The minimum supported focal length of the camera
    optional float maximum_focal_length = 2;  // The maximum supported focal length of the camera
    repeated float   focal_length_steps = 3;  // The decimal focal length steps between the whole number focal lengths
}

enum WSDK_LensGroupType {
    Unknown = 0;
    Mixed   = 1;
    Fisheye = 2;
}

message WSDK_LensGroup {
    optional WSDK_LensGroupType group_type = 1;
    repeated WSDK_Lens              lenses = 2;
}

message WSDK_LensList {
    repeated WSDK_LensGroup        lense_groups = 1;  // Groups of specific lenses (fisheye, curated, etc.)
    optional WSDK_LensParameters     parameters = 2;  // Returned when list describes all possible lenses
    optional int32           max_curated_lenses = 3;  // Maximum curated lenses the camera can store
}

message WSDK_RequestCuratedLensList {
    optional bool register_for_updates = 1;  // Register for active lens / curated list updates
}

message WSDK_RequestAddUserLens {
    optional int32                   lens_id = 1;  // Lens id of a supported lens to add
    optional float  custom_lens_focal_length = 2;  // The focal length of the user's lens to add
}

message WSDK_RequestDeleteUserLens {
    optional int32 lens_id = 1;  // ID of the lens to delete from the curated list
}

message WSDK_RequestSetActiveLens {
    optional int32 lens_id = 1;  // ID of the lens to set active from the curated list
}
```

### BLE Commands

```protobuf
enum WSDK_EnumCmdId {
    ...
    WSDK_CMD_ID_REQUEST_DELETE_DETACHABLE_LENS          = 0x5D;  // Delete a lens from the curated list
    WSDK_CMD_ID_REQUEST_ADD_DETACHABLE_LENS             = 0x5E;  // Add a detachable lens to the curated list
    WSDK_CMD_ID_REQUEST_SET_ACTVE_DETACHABLE_LENS       = 0x5F;  // Set active detachable lens
    ...
}
```

### BLE Queries

```protobuf
enum WSDK_EnumQueryId {
    ...
    WSDK_QUERY_ID_REQUEST_GET_DETACHABLE_LENS_SUPPORTED_LIST = 0x6A;  // Returns WSDK_LensList
    WSDK_QUERY_ID_REQUEST_GET_DETACHABLE_LENS_CURATED_LIST   = 0x6B;  // Returns WSDK_LensList
    ...
}
```

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

One shared model for both factory and custom lenses:

```protobuf
message WSDK_Lens {
    optional int32  id           = 1;  // Opaque handle, assigned by camera
    optional string name         = 2;  // Display label, always populated by camera
    optional float  focal_length = 3;  // Always populated
    optional bool   active       = 4;  // True = currently active on camera
}
```

### Create (two methods)

These are two real user actions — entering a number vs picking from a chooser — not just moving the firmware duality to a higher layer.

```protobuf
// 1. Add Catalog Lens to Curated List
message WSDK_RequestAddCatalogLensToCuratedList {
    optional int32 lens_id = 1;  // id from the supported catalog response
}

// 2. Add Custom Lens to Curated List
message WSDK_RequestAddCustomLensToCuratedList {
    optional float focal_length = 1;
}

// Response for both: ResponseGeneric
// Note! This may need to return an ID depending on the UI flow
```

### Read (two methods)

```protobuf
// 1. READ curated list
message WSDK_RequestGetCuratedList {
    optional bool register_for_updates = 1;
}
message WSDK_ResponseLensList {
    repeated WSDK_Lens lenses = 1;
}

// 2. READ supported catalog
message WSDK_RequestGetSupportedLensCatalog { }
message WSDK_ResponseSupportedLensCatalog {
    repeated WSDK_Lens           lenses = 1;  // Named catalog lenses
    optional WSDK_LensParameters  range = 2;  // Constraints for custom focal length input
}
message WSDK_LensParameters {
    optional float min  = 1;
    optional float max  = 2;
    optional float step = 3;
}
```

### Update

```protobuf
message WSDK_RequestSetActiveLens {
    optional int32 lens_id = 1;  // id from curated list
}
```

### Delete

```protobuf
message WSDK_RequestRemoveLensFromCuratedList {
    optional int32 lens_id = 1;  // id from curated list
}
```
