# Vigil for Home Assistant

Brings [Vigil](https://vigil.altair571.app) (an NVR plugin for Scrypted) into Home Assistant:

- **Two dashboard cards**: recent events as tiles, and a camera card. Tapping either opens a popup that plays the event's clip, plays a visit across several cameras in order, or shows live video.
- **Entities** per Vigil camera: a snapshot camera, and an event entity that fires for every new Vigil event so automations can react.

Nothing streams until you ask: the cards show stills and event pictures, refresh every 10 seconds, and only while they are on screen.

Needs Home Assistant 2025.1 or newer and the Vigil plugin (`@altair571/nvr`) on your Scrypted server.

<!-- ha-marketplace:exclude-start -->
## Install

[![Open your Home Assistant instance and open this repository in the Marketplace.](https://my.home-assistant.io/badges/hacs_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=Altair571&repository=vigil-home-assistant&category=integration)

**Home Assistant 2026.11 or newer.** Use the button above, or:

1. Go to **Settings → Marketplace** and choose **Add from link**.
2. Enter `Altair571/vigil-home-assistant` and choose **Add**. The first time, the Marketplace asks you to connect GitHub; that is how it reads repositories added from a link.
3. Open **Vigil** in the list and choose **Install**.

**Older Home Assistant, with HACS.** In HACS, open the menu and choose **Custom repositories**. Add `https://github.com/Altair571/vigil-home-assistant` with the type **Integration**, download **Vigil**, and restart Home Assistant.

**By hand.** Download `vigil.zip` from the [latest release](https://github.com/Altair571/vigil-home-assistant/releases/latest), unpack it into `custom_components/vigil` in your Home Assistant configuration folder (the ZIP holds that folder's contents), and restart Home Assistant.

Updates then show up under **Settings → Updates**, like Home Assistant's own.
<!-- ha-marketplace:exclude-end -->

## Set up

**Settings → Devices & services → Add integration → Vigil** ([open it in your Home Assistant](https://my.home-assistant.io/redirect/config_flow_start/?domain=vigil)), then enter:

| Field | |
|---|---|
| Scrypted address | For example `https://192.168.0.50:10443`. A bare address such as `192.168.0.50` works too. |
| Username, password | A Scrypted user. |
| Verify SSL certificate | Leave off for Scrypted's own self-signed certificate (the default). |

The password is used once, to get Scrypted's sign-in token for that user, and is not stored. Home Assistant keeps the token (in its own configuration storage, like every integration's credentials), and uses it to get a short-lived session from Scrypted, renewed as needed. This is the same sign-in the Vigil iPhone app uses.

With certificate verification off, the certificate the server shows when you add it is remembered, and from then on only a server showing that certificate is trusted. If the certificate changes, or the password does, Home Assistant asks you to sign in again.

**Home Assistant sees what that Scrypted user may see.** Vigil has per-user access rules (which cameras, live view, recordings), so a restricted Scrypted user made for Home Assistant is a good idea. Everyone who can use your Home Assistant dashboards then sees what that user sees. Which events count as reviewed is also that user's, shared by everyone in Home Assistant.

## Vigil in the sidebar

**Vigil** in the sidebar opens Vigil's own app inside Home Assistant: no second sign-in, no certificate warning.

- Everyone who opens it acts as the integration's Scrypted user, settings included if that user is an administrator. Use a restricted user, or **Configure → Only for Home Assistant administrators**. **Show Vigil in the sidebar** turns it off.
- You count as home only when reaching Home Assistant by a home address with no proxy or tunnel between.
- The app runs from Home Assistant's own address, so Home Assistant trusts your Scrypted server like a custom card.

## Entities

For each Vigil camera, one device with:

- `camera.<name>_snapshot`: Vigil's latest picture. No stream, so it costs nothing until something asks for a picture.
- `event.<name>_detection`: fires `detected` once for each new Vigil event (checked every 10 seconds). Attributes: `kind` (person, face, vehicle, animal, package, doorbell, motion…), `label` (a recognised face's name, a number plate), `camera`, `camera_id`, `event_id`, `start`, `end`, `score`, `zones`, `source`, and `thumbnail` (the picture's path in Home Assistant).

Events that already existed when Home Assistant started don't fire.

```yaml
automation:
  - alias: Porch light when someone is at the front door
    triggers:
      - trigger: state
        entity_id: event.front_door_detection
        not_from: unavailable
    conditions:
      - condition: template
        value_template: "{{ trigger.to_state.attributes.kind in ['person', 'face'] }}"
    actions:
      - action: light.turn_on
        target:
          entity_id: light.porch
```

## Cards

The cards load with Home Assistant's pages. A page that was already open when you added the integration (or updated it) doesn't have them yet and says "Custom element doesn't exist": refresh the browser.

The integration serves the cards and loads them itself; there is no resource to add. Both appear in the card picker ("Vigil events", "Vigil camera") with a visual editor.

### Events

```yaml
type: custom:vigil-events-card
title: Recent
```

```yaml
type: custom:vigil-events-card
layout: grid
limit: 12
kinds: [person, face, package]
cameras: [Front Door, Driveway]
camera_entities:
  Front Door: camera.front_door   # played when "Live" is tapped in the popup
```

```yaml
type: custom:vigil-events-card
style: plain        # the picture and its time, nothing else
layout: grid
columns: 3
rows: 2             # two rows at a time; swipe for the rest
limit: 30
```

Up to `limit` square tiles, newest first: the picture, what it was (a recognised face's name or a number plate when there is one), the camera, and how long ago. A visit seen on several cameras is one tile showing a picture from each, with the number of cameras. Tiles you haven't opened yet have a coloured ring.

| Option | Default | |
|---|---|---|
| `title` | none | A heading, with the number of new events and a "mark all as reviewed" button. |
| `layout` | `row` | `row` (one row that scrolls sideways) or `grid`. |
| `style` | `detailed` | `detailed` puts what it was, the camera and how long ago over the picture. `plain` shows the picture alone with its time underneath. |
| `columns` | automatic | Grid only: a fixed number of columns (3 when `rows` is set). |
| `rows` | none | Grid only: show this many rows at a time and swipe sideways for more, with a dot per page. Without it the grid shows everything at once. |
| `limit` | `20` | The most events to show, 1 to 50. With `rows`, the pages hold up to this many between them. |
| `transparent` | `true` for `plain`, otherwise `false` | No card behind the tiles: they sit straight on the dashboard. |
| `player` | `clip` | What a tapped event opens. `clip` plays the event's clip. `vigil` shows Vigil's own player at that moment, with its timeline, controls and the day's other events; it needs "Vigil in the sidebar" (below) switched on, and plays the clip if that isn't available. |
| `kinds` | all but motion | A list of kinds, or `all`. |
| `cameras` | all | Vigil camera names or ids, or the integration's camera entities. |
| `episodes` | `true` | `false` shows each camera's sighting as its own tile. |
| `entry` | the only one | The Vigil server's name, when more than one is set up. |
| `camera_entities` | none | Vigil camera → a Home Assistant camera with a stream, for "Live" in the popup. |
| `vigil_url` | the address Home Assistant uses | Where "Open in Vigil" goes, for example your Scrypted Cloud address. `false` hides the button. |

### Camera

```yaml
type: custom:vigil-live-card
entity: camera.front_door_snapshot
camera_entity: camera.front_door   # optional: a camera with a stream, for the popup
show_events: true
```

The camera's latest picture (a new one every 10 seconds while the card is on screen; the card itself never streams), its name, and what it last saw. `show_events` adds a strip of its latest events.

| Option | Default | |
|---|---|---|
| `entity` | | One of the integration's snapshot cameras. Or use `camera` with a Vigil camera name or id. |
| `camera_entity` | none | A Home Assistant camera with a stream (WebRTC or HLS), played in the popup. |
| `name` | the camera's | |
| `show_events` | `false` | The strip of latest events. |
| `limit` | `8` | How many events in the strip. |
| `kinds` | all but motion | As above. |
| `aspect_ratio` | the picture's | For example `16:9`. |
| `refresh` | `10` | Seconds between pictures (5 or more). |
| `entry`, `vigil_url` | | As above. |

### The popup

- **An event**: its clip, from 3 seconds before to 3 seconds after, with when, where and how long. Buttons: **Live**, **Open in Vigil**.
- **A visit on several cameras**: each camera's part plays in turn, with a strip showing the parts and where playback is. Click a part, or press ← and →, to jump.
- **Live**: Home Assistant's own player for the camera in `camera_entity` (or `camera_entities`), so it plays however that camera streams in Home Assistant. Without one, the popup shows Vigil's picture, refreshed every few seconds.

Opening an event or a visit marks it reviewed in Vigil. The popup fills the screen on a phone; Escape, the close button or a click outside closes it.

## Good to know

- **Video formats.** Clips are the camera's own video. A browser that can't play H.265 shows a message instead; "Open in Vigil" plays it there.
- **Safari and the iPhone app** ask for video in pieces, so Home Assistant fetches the whole clip first (up to 120 MB, kept in memory for three minutes). Other browsers play it as it arrives.
- **Away from home.** Vigil can close a camera to people away from home (its privacy settings). The cards and the sidebar tell Vigil whether the person looking is away: you count as home only when you reach Home Assistant by a home address with no proxy or tunnel between. Home Assistant's own camera entity can't tell who is looking, so a camera closed to people away from home has no picture through `camera.<name>_snapshot`, at home either; the Vigil cards still show it at home.
- **Older Vigil.** Without the newer "recent events" list, the cards fall back to the plain event list: single events only, from the last day, without reviewed rings.
- **What Home Assistant can reach.** Its pages under `/api/vigil/` need a signed-in Home Assistant user, and only ever ask Vigil for a fixed set of things (the camera list, recent events, an event's picture, a camera's snapshot, a clip of at most 10 minutes, marking events reviewed) for cameras Vigil listed. They are not a general way into Scrypted.

## Help

Questions and problems: [open an issue](https://github.com/Altair571/vigil-home-assistant/issues) or write to support@vigil.altair571.app. More about Vigil: [vigil.altair571.app](https://vigil.altair571.app).

Vigil for Home Assistant is closed source: see [LICENSE](LICENSE). This repository holds its releases; each one carries the integration as `vigil.zip`.
