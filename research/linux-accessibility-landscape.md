# Linux accessibility landscape, October 2026

The moving parts behind the current argument about accessibility on the free desktop, written
out in plain language, with sources. Companion to
`touch-input-and-assistive-technology.md`.

It came out of a Mastodon thread on 1 October 2026 in which Florian Beijers (zersiax) said
accessibility is better than a decade ago but nowhere near good, and thirty-four replies later
the conversation had turned into what is technically missing on Linux.
https://cupoftea.social/@zersiax/117362789935375556

## Who is working on what

- **tyrylu, Lukáš Tyrychtr** — GNOME and Orca accessibility. Reports running accessible Ghidra on
  Arch after working out the Java accessibility setup, and that the Super key announcement is
  fixed in GNOME 51.
- **MikeGorse, Mike Gorse** — GNOME accessibility, including the accessibility bus. Says the real
  problem is finding more developers.
- **menelion, André Polykanine** — accessibility auditor and translator. His bar for a usable
  Linux desktop: a blind person can work, rest and tinker without sighted help, and a real screen
  reader must be scriptable with robust braille.
- **bram, Bram Duvigneau** — long-time accessibility developer, skeptical of building a reader over
  the mixture of window managers and old APIs.
- **esoteric_programmer** — the technical plan in the thread. Says the important work is Wayland
  protocols and assistive technology integration rather than per-desktop fixes; names portals as
  the right abstraction; says the current approach is a hack; and says the first thing is to
  revive the Newton branch. Also says some of it cannot be fixed with code, because the design is
  the problem.
- **Joanmarie Diggs** — Orca, for years, largely alone.
- **Matt Campbell** — Newton, and the AccessKit work behind the GTK integration.
- **Peter Hutterer** — libei, and the portal integration that matters for touch.
- **Devin Prater, pixelate** — raised the Super key announcement and contracted Grade 2 braille
  being blocked by BRLTTY's console focus, and offered a touch-screen laptop for the touch work.

## The layers

- **AT-SPI** — the old pipe, designed in the early 2000s. Pull based: the reader asks for one
  property at a time. Runs on its own D-Bus, the accessibility bus, and sandboxed Flatpak
  applications need an exception to reach it. https://freedesktop.org/wiki/Accessibility/AT-SPI2/
- **Push architecture** — what Chromium and Firefox do internally: the application pushes a whole
  accessibility tree plus updates, and the reader answers its own questions from a local copy. It
  survives a hung application, because the last tree is still there.
- **Newton** — the push-based, Wayland-native accessibility architecture, named after Newton,
  Massachusetts, home of the Carroll Center for the Blind. Applications push tree updates through
  a draft Wayland protocol; readers connect to the compositor over D-Bus, so the compositor, not
  the application, decides what has focus. Prototyped in 2024 with funding from the Sovereign Tech
  Fund, never upstreamed.
  https://blogs.gnome.org/a11y/2024/06/18/update-on-newton-the-wayland-native-accessibility-project/
  https://lwn.net/Articles/971541/
- **AccessKit** — the Rust library toolkits use to produce that tree. GTK uses it on Windows and
  macOS today; the GTK integration is still a draft merge request, and the GTK hackfest in early
  2026 said they see no movement in it. https://lwn.net/Articles/1058024/
  https://gitlab.gnome.org/GNOME/gtk/-/merge_requests/7401
- **AT-SPI3** — a maintainer in the AccessKit discussions says he has started it, inspired by the
  Newton prototype. https://github.com/AccessKit/accesskit/issues/670
- **Orca and Odilia** — Orca is the desktop reader, largely one person's work. Odilia is a newer
  Rust reader, often named as a second consumer of the new protocols.
- **The two a11y D-Bus interfaces** — `org.freedesktop.a11y.KeyboardMonitor` and
  `org.freedesktop.a11y.PointerLocator`, at `/org/freedesktop/a11y/Manager` on the session bus.
  Orca uses them for keyboard monitoring and mouse review. Mutter 48 and KWin 6.3.90 implement
  them; it is an informal cross-desktop agreement, not a specification.
  https://discourse.gnome.org/t/my-accessibility-stack-and-the-future-on-wayland/35230
- **The security hole in those interfaces** — a process can claim the bus name
  `org.gnome.Orca.KeyboardMonitor`, and because the compositor treats owning that name as proof
  that the caller is Orca, it can then receive raw keystrokes, passwords included. Confirmed on
  both KWin and Mutter. https://linnemanlabs.com/posts/hello-my-name-is-orca/
  https://invent.kde.org/plasma/kwin/-/merge_requests/9825

## Portals

`xdg-desktop-portal` is the doorman for sandboxed applications: an application asks the desktop
for something, and the user is asked to allow it. Assistive technology needs more access than any
other kind of application, which is why the argument keeps returning here: handing that access out
by matching a bus name is what produced the keyboard hole above.

Portals that exist and matter here:

- **GlobalShortcuts** — application shortcuts, and the starting point for a reader's own keys.
- **RemoteDesktop** — screen capture, which is how an OCR-style reader or a magnifier would work.
- **InputCapture** — hands a client keyboard, pointer or touchscreen events through libei, with a
  permission store deciding what is allowed silently and what needs a prompt. Session persistence
  since portal 1.21.0. This is the closest thing to touch support that exists.
  https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.InputCapture.html

Portals that are only proposed:

- **`org.freedesktop.portal.AT.Shortcuts`** — letting a reader bind and unbind keys freely,
  including odd ones like Caps Lock plus a letter. https://github.com/flatpak/xdg-desktop-portal/issues/1046
- **An accessibility portal** — for handing out access to the accessibility bus, and formalising
  the two a11y interfaces so they sit behind a permission prompt and readers can ship on Flathub.
- **Pointer position** — a request for reading where the pointer is, beyond which window it is in.
  https://github.com/flatpak/xdg-desktop-portal/issues/880

## Related pieces worth knowing about

- **BRLTTY and speakup** — the console braille and console speech side, which is the source of the
  contracted Grade 2 braille problem: BRLTTY is aimed at the console, so the graphical session does
  not get the contracted tables the way a user would expect.
- **Java accessibility** — Java applications need an access bridge configured by hand, through
  `accessibility.properties` or environment variables. On Arch a missing symlink made it worse and
  OpenJDK behaviour changed underneath it. On Windows, NVDA and JAWS ship their own bridge, which
  is why it feels like it works there. Fixing this is what made Ghidra accessible on Arch.
- **webAIM Million** — the annual audit of one million home pages, cited in the thread as the
  evidence that the web has not fixed itself over twenty years.

## Things that could be worked on, from the thread

1. **Revive the Newton branch** — `pop-os/orca` at `newton_noble`, an Orca fork with Newton support
   and Debian packaging. Last commits June and August 2024, so about two years stale.
   https://github.com/pop-os/orca/tree/newton_noble
2. **Draft the accessibility portal** — the interface that would replace the two a11y D-Bus
   interfaces, covering key monitoring and pointer location, with the argument that name based
   authorisation is a security hole. Lands on issue 1046 above.
3. **The touch gesture gap** — see `touch-input-and-assistive-technology.md`.
4. **The Java accessibility recipe** — write down the exact setup for Ubuntu and WSL, where the
   missing symlink and the OpenJDK change bite, so Java applications including Ghidra can be made
   accessible without guesswork.
