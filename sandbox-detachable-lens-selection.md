# Sandbox Detachable Lens Selection Interface

## Purpose

The purpose of this wiki is to describe the Sandbox lens focal length selection interface between camera (BOSS) and app (WSDK). With the introduction of user-provided detachable lenses, the user has a need to curate a custom list of the available lenses that they have in their possession. They will then be able to quickly select from that list the lens that is currently in use.

## Camera UI Design

The design of the camera can be found here:

[Figma: SB Specifications](https://www.figma.com/design/7Ji5sTYtZwGbwrleclUR7P/SB-Specifications?node-id=32261-111976&t=6wPurxk8dpZAmSpy-4)

![Camera UI Design](https://raw.githubusercontent.com/tcamise-gpsw/redraft-test-repo/demo/lens-selection/images/image-20260624-000140.png)

## App UI Design

The design for the app is currently here, though as of 7-1-2026 it is missing the curated list selection screen:

[Figma: KAT Quik App](https://www.figma.com/design/2DYSCNn09DslOsWHCK5Gu5/KAT-Quik-App?node-id=919-182897&m=dev)

![App UI Design](https://raw.githubusercontent.com/tcamise-gpsw/redraft-test-repo/demo/lens-selection/images/image-20260701-225505.png)

## Interface Requirements

- The app must have a method of retrieving the curated list of lenses, including the active lens, from the camera.
- The app must have a method of being notified that there have been changes to the curated list of lenses.
- The app must have a method of setting a lens from the curated list as active on the camera.
- The app must have a method to get the complete list of supported lenses from the camera.
- The app must have a method to add a lens from the supported list to the curated list on the camera.
- The app must have a method to delete a lens from the curated list on the camera.
- The app must have a way of knowing that the camera supports the operations listed above.

## FW Design

The FW design can be found here: Sandbox FW - Lenses

## Proposal 1

The app will know that the camera supports the detachable lenses feature through the use of new settings.json command ids.

Each of the above will have a related BLE command with protobuf data support.

### Command Specification

| Command | Params | Response | Description |
| --- | --- | --- | --- |
| `WSDK_QUERY_ID_REQUEST_GET_DETACHABLE_LENS_SUPPORTED_LIST` | None | `WSDK_LensList` | Query to get the full list of lenses supported by the camera. The object contains 2 types of data: a range of mm describing the lens **parameters**, and a fixed list of specific **lenses**. Clients will use this data to create the 2 separate selection mechanisms available to the user — the numeric selection and the fisheye lens selection. Data in the **lenses** object will only have the `supported_lens_id` and `detachable_lens_name` fields supplied. |
| `WSDK_QUERY_ID_REQUEST_GET_DETACHABLE_LENS_CURATED_LIST` | `WSDK_RequestCuratedLensList` | `WSDK_LensList` | Query to get the list of lenses added to the camera's curated list for quick selection. The **register_for_updates** param specifies whether the client requests dynamic updates for active lens and curated list changes. Return objects have an `id`, plus either a `focal_length` (mm) or a `supported_lens_id`. If a `supported_lens_id` is provided, `detachable_lens_name` will also be supplied. The current active lens will have `active` set to `true`. |
| `WSDK_CMD_ID_REQUEST_ADD_DETACHABLE_LENS` | `WSDK_RequestAddUserLens` | `ResponseGeneric` | Add a lens. Use the supported list response to fill the request with either a `supported_lens_id` or a `custom_lens_focal_length`. If both values are supplied, an error is returned. |
| `WSDK_CMD_ID_REQUEST_DELETE_DETACHABLE_LENS` | `WSDK_RequestDeleteUserLens` | `ResponseGeneric` | Remove a lens from the camera's curated list. Supply the `lens` object retrieved from the curated list query. |
| `WSDK_CMD_ID_REQUEST_SET_ACTIVE_DETACHABLE_LENS` | `WSDK_RequestSetActiveLens` | `ResponseGeneric` | Set the described `lens` active on the camera. The lens must be from the curated list query response. |

## Proposal 2

I think it is important to apply proper interface segregation and not have the app ingest information / complexity that it has no use for. The main example of this from Proposal 1 is that firmware stores catalog lenses as an enum and custom lenses as a float, serializes both representations directly, and makes the app figure out which one it's looking at.

The app really only needs:

- Some type of `ID` to identify a lens for operations
- A displayable `name` string
- Some type of state, currently only `isActive`
- Maybe, if there is a need to operate on it, a float `focalLength`

Anything else is firmware implementation details that serves no purpose in an app interface.

If the camera resolves the factory/custom distinction at the boundary and always returns a uniform lens object, then from the app's perspective this is simple CRUD — with two Create methods since there are two user inputs.

### Domain Model

The one shared domain model for both factory and custom lenses contains all (and only) app-relevant information.

### Operations

| Operation | Method | Description |
| --- | --- | --- |
| **Create (factory)** | Add from supported list | User picks from a chooser |
| **Create (custom)** | Add by focal length | User enters a number |
| **Read (supported)** | Get supported list | Full catalog of available lenses |
| **Read (curated)** | Get curated list | User's curated list with active state |
| **Update** | Set active lens | Select a lens from the curated list |
| **Delete** | Remove from curated list | Remove a lens from the user's list |

Note that these two Create methods are not just moving the firmware model duality to a higher layer. These are two real user actions: entering a number vs picking from a chooser.
