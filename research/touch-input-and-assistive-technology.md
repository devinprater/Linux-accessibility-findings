# Does any Linux desktop give assistive technology access to touch input?

First pass, 10 October 2026. Written by Devin Prater with the agent, from public sources and
from reading the source of Orca and the portal definitions.

See `how-other-readers-do-touch.md` for how the readers that do have touch are built, and what
they suggest for Linux.

## The question

A blind user with a touch screen on the Linux desktop has no equivalent of touch exploration,
which is how screen readers work on Android and iOS: drag a finger over the screen and hear
what is under it, and use one, two and three finger gestures to move, scroll and activate.
This document asks what exists today on Linux, and what is missing. It started from a
Mastodon thread about the state of Linux accessibility, in which touch gestures were named as
one of the gaps in the Wayland accessibility work.

## Answer in short

No. As of October 2026 no Linux desktop exposes touch input to assistive technology in a
usable way, and no screen reader on Linux has touch support.

Orca, the screen reader on the Linux desktop, has no touch handling in its source at all, and
its maintainer said in 2021 that something would first have to be added to AT-SPI. AT-SPI
itself has no touch events: it can synthesize keyboard and mouse events, and that is all.

The transport, however, already exists, and this is the interesting part. libei carries touch
events, including gestures, and the xdg-desktop-portal InputCapture portal can hand a permitted
client the touchscreen of the session. The portal maintainer wrote in July 2026 that assistive
technology is one of the use cases for that plumbing that has not been scoped yet. So the
missing piece is not a transport, it is a reader that uses one.

## What exists today

### Orca, the Linux screen reader

- Orca works through AT-SPI, with speech and braille, and its documented navigation is
  keyboard, flat review and mouse review. There is no touch mode and no gesture set.
  https://help.gnome.org/orca/introduction.html
- Joanmarie Diggs, the Orca maintainer, in February 2021: "I'm afraid Orca does not have any
  touch-specific support, and I believe something would need to be added to AT-SPI2 in order
  for it to be added."
  https://lists.gnome.org/archives/orca-list/2021-February/msg00193.html
- Checked today: a code search over GNOME/orca for "touch" returns 9 files, none of which
  handle touch input. They are the speech dispatcher, SSML handling, settings documentation,
  a French translation and tests. A search of Orca's NEWS for touch returns nothing. So the
  2021 statement still holds as far as the code shows.

### AT-SPI, the accessibility pipe

- AT-SPI is a protocol over D-Bus. Applications describe their widget tree, and assistive
  technologies read it. https://freedesktop.org/wiki/Accessibility/AT-SPI2/
- The client library can synthesize events, and only two kinds: `generate_keyboard_event` and
  `generate_mouse_event`, the latter described as synthesizing "a mouse event at a specific
  screen coordinate". There is no touch event and no touch device interface.
  https://docs.gtk.org/atspi2/
- The one input-facing piece of AT-SPI on the compositor side is a pair of D-Bus interfaces,
  `org.freedesktop.a11y.KeyboardMonitor` and `org.freedesktop.a11y.PointerLocator`, used by
  Orca for keyboard monitoring and for finding the window under the pointer. There is no touch
  equivalent. https://discourse.gnome.org/t/my-accessibility-stack-and-the-future-on-wayland/35230

### The transport that does exist: libei, and the InputCapture portal

- libei is a transport for logical input events, with a sender mode for emulating input and a
  receiver mode for capturing it. It carries touch: `EI_EVENT_TOUCH_DOWN`,
  `EI_EVENT_TOUCH_MOTION`, `EI_EVENT_TOUCH_UP` and `EI_EVENT_TOUCH_CANCEL`, on an
  `ei_touchscreen` interface. https://libinput.pages.freedesktop.org/libei/api/group__libei.html
- libei also carries gestures. `EI_DEVICE_CAP_GESTURES` covers swipe, pinch and hold, and
  stylus events were added in version 1.7. Same page.
- The xdg-desktop-portal InputCapture portal exposes exactly this to sandboxed applications,
  with a capability bitmask whose defined values are 1 keyboard, 2 pointer and 4 touchscreen.
  Access is controlled by a permission store, and since portal 1.21.0 a session can be
  remembered instead of re-prompting.
  https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.InputCapture.html
- It is used in practice by remote-desktop style tools, for example Deskflow, and the
  permission store is set per application with `flatpak permission-set inputcapture`.
  https://github.com/flatpak/xdg-desktop-portal/pull/1768
- Peter Hutterer, who wrote libei, in July 2026: the portal plumbing is in place, and the use
  cases "aren't fully scoped yet (assistive technologies, virtual keyboards, touchpads, etc?)
  but the piping is there and ready to be (ab)used". He names text, gestures and tablet
  support as the next pieces of work.
  http://who-t.blogspot.com/2026/07/libei-integrations-in-xdg-remotedesktop.html

### Linux on phones

- postmarketOS keeps an accessibility page, and one of its issues asks directly why blind users
  are excluded. The answer from the project was that a screen reader must be implemented in the
  user interface, that Orca did not work in Phosh when it was last tested, and that there is a
  shortage of contributors who know the area.
  https://gitlab.com/postmarketOS/pmaports/-/work_items/2321
- Phosh's development news mentions accessibility work such as setting accessible roles for
  screen readers, and touch fixes, but nothing about a touch-driven reader.
  https://phosh.mobi/posts/development-news-2026-09/
- postmarketOS v26.06 describes vibration on boot as part of a larger effort to make the system
  more accessible, and points to a chat for people interested in that work.
  https://postmarketos.org/blog/2026/06/21/v26.06-release/

## What is missing

1. No reader that consumes touch. The mechanism to receive touchscreen events exists and is
   permitted through a portal; nothing that a blind user would run uses it.
2. No mapping from a finger position to the accessible item under it. A reader would have to
   take a coordinate from the compositor and find the element whose bounds contain it, in
   window-relative coordinates, then speak it.
3. No gesture recognition for reader gestures. The events arrive, but the meaning of a two finger
   swipe or a double tap is for the reader to define, and libei does not decide that.
4. No agreed way to stop the application reacting to a gesture the reader has taken. Touch
   exploration requires swallowing the touch; the InputCapture portal does redirect input to the
   client, which is the right shape, but its implications for assistive technology have not been
   worked through in public.
5. No discussion on record of what this should look like, which is why this document exists.

## What a touch-capable reader would plausibly need

This is a design sketch, not a proposal. It follows the shape of the existing portal work.

- Request the touchscreen through the InputCapture portal, so the permission is a user
  decision in the same place as other portal permissions, and the reader can ship as a
  Flatpak.
- Receive touch events as coordinates through libei in receiver mode.
- Map coordinates to the accessibility tree, then speak the item under the finger. The tree
  comes from the accessibility stack, not from libei.
- Recognize reader gestures itself, with a data file of gestures rather than code, so users can
  add and share them, which is the same argument raised on the Orca list in 2021.
- Activate items through the accessibility action interface rather than by synthesizing a tap,
  which is what the AT-SPI documentation recommends for clients, and which keeps the reader
  from fighting the application over the same input.
- Decide what happens to the gesture the reader took, which is the part that needs the most
  discussion, because the alternative, letting both the reader and the application act, would
  be unusable.

## What to test next

- Does the InputCapture portal deliver touchscreen events to a plain client at all, on GNOME
  and on KDE? Nobody appears to have written this down.
- What exactly arrives: raw coordinates, or something already mapped to a surface?
- Does a captured touchscreen stop the application receiving touches, and can that be paused
  and resumed quickly enough for touch exploration?
- Is there any compositor that exposes touch to assistive technology outside the portal?

## What was checked, and how

Checked by reading the sources above, dated 10 October 2026:

- Orca's documentation, maintainer statement, and source code, including a code search for
  touch handling and a search of its release notes.
- The AT-SPI client API documentation for event synthesis, and the accessibility bus
  documentation.
- The libei API documentation for touch and gesture events, and the InputCapture portal
  documentation for its capability bitmask and permission store.
- The libei author's July 2026 write-up on the portal integration.
- The postmarketOS issue on blind users, Phosh's development news, and the postmarketOS
  release notes.

Not verified, and worth being honest about:

- No Linux desktop was run for this document. Everything here is documentation and source
  reading, so the portal behaviour for touch is inferred from documentation rather than
  measured. A measured test is the obvious next step, see `experiments/`.
- Odilia, the Rust screen reader, was not examined. It may have touch work that this document
  does not know about.
- GNOME Shell's mobile variant and Plasma Mobile were not examined in detail.
- AT-SPI3, which is being started by a maintainer in the AccessKit discussions, was not
  examined for touch plans.
