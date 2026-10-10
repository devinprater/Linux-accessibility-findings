# How other screen readers do touch, and what a Linux reader can borrow

Written 10 October 2026. Companion to `touch-input-and-assistive-technology.md`, which
establishes that no Linux desktop gives assistive technology touch input today. This document
asks the opposite question: for the readers that do have touch, how is it built, and which parts
of that design could a Linux reader reuse?

Sources are the readers' own source code where it is open, and their documentation where it is
not. See the end for what was checked and what was not.

## The six patterns that every implementation shares

1. **The platform owns the touch pipeline.** No reader talks to the touch driver, or to
   /dev/input. Windows hands pointer messages to a program that registers for them, and Android
   hands motion events to the accessibility service that asked for them. The reader never
   competes with the system for input.
2. **The reader either classifies gestures itself or receives them already classified.** Windows
   gives raw pointer messages and NVDA does its own tracking. Android detects the standard
   gestures in the framework and delivers them by name, while also letting the service see the
   raw events and decide for itself.
3. **Exploration and activation are two different paths.** Touches are turned into reads, by
   hit testing what is under the finger. Activation goes through the accessibility action API,
   not by replaying a tap. Android's controller does exactly this: it tries
   `ACTION_CLICK` on the item with accessibility focus first, and only falls back to simulating
   a click at that location if the action is refused.
4. **Hit testing uses a platform element-at-point call.** This is the primitive that makes
   exploration possible, and Linux already has one: AT-SPI's Component interface has
   `GetAccessibleAtPoint(x, y, coord_type)`, alongside `Contains` and `GetExtents`.
5. **Every implementation has a pass-through, so games and apps can still get raw touch.**
   Android does it with named regions, JAWS and NVDA with a toggle, VoiceOver with a direct
   touch typing mode. A reader that always swallows touch is unusable for games.
6. **Every implementation has modes, changed by a dedicated gesture, and gesture sets that are
   configurable rather than hard coded.** NVDA has touch modes cycled with a three finger tap,
   JAWS cycles three modes with a two finger rotate, VoiceOver has the rotor, Narrator changes
   views. The reasons are the same everywhere: the same gesture has to mean different things in
   text, in a list and in a game.

## Windows: one mechanism, three vocabularies

### The mechanism

Windows delivers touch to a program as pointer messages, `WM_POINTERDOWN`, `WM_POINTERUPDATE`,
`WM_POINTERUP` and so on, with the details in a `POINTER_TOUCH_INFO` structure. A program can
ask to receive all of the touch input rather than only what lands on its own windows, by calling
`RegisterPointerInputTarget`. That is what NVDA does, and it is what lets the reader work over
any application instead of only its own window.

In NVDA this is `source/touchHandler.py`: the constants for the pointer messages are at the top
of the file, and the handler thread registers itself at line 473 with
`user32.RegisterPointerInputTarget(self._touchWindow, PT_TOUCH)`.

### NVDA, from its source and its manual

- **Toggle.** Touch support is off by default and toggled with `NVDA+control+alt+t`, or from the
  Touch Interaction settings page. While it is off, all touch goes straight through to the
  application.
- **Exploration.** Place one finger anywhere on the screen and it announces what is there; keep
  the finger down and move it to read what the finger passes over. In the source this is the
  hover family of actions, `hover down`, `hover` and `hover up`, and the manual describes the
  behaviour as exploring the screen.
- **Gestures.** Taps, with the finger count multiplied, so a two finger tap and a four finger tap
  are both real gestures, and repeated taps become multi-taps, up to quadruple. Flicks in four
  directions, again with finger counts. Pinch in and pinch out. Tap and hold. Sequential flicks,
  which are two flicks in quick succession, either by lifting and flicking again or by one
  continuous swipe that sharply changes direction.
- **Modes.** A three finger tap cycles touch modes, and the mode is part of the gesture identity,
  for example `ts(mymode):flickRight`. Browse mode has its own touch subset: flick up and down
  changes the element type, flick left and right moves between elements of that type.
- **Hit testing.** NVDA finds the object under a point with `objectFromPoint` in
  `source/NVDAObjects/__init__.py`, which goes through the platform accessibility APIs rather
  than through the touch message itself.

### Narrator

Documented behaviour, one finger to read what is under the finger, double tap to activate, and
then a large gesture vocabulary:

- One finger: drag reads under the finger, flick left and right moves to the next and previous
  item, flick up and down changes the Narrator view.
- Two fingers: tap stops reading, double tap shows the context menu, triple tap dismisses a
  context menu, swipe scrolls in any direction.
- Three fingers: tap changes verbosity, swipe left and right tabs forward and back, swipe up
  reads the current window, swipe down starts reading.
- Four fingers: tap shows commands for the current item, double tap opens search, three taps
  shows all commands, swipe up and down toggles semantic zoom, swipe left and right moves the
  cursor to the beginning or end of a unit.
- A hold with one finger and then a tap with a second finger activates the primary action, the
  same shape as a split tap.

The view concept is the same idea as NVDA's modes: one finger flicks up and down change what the
left and right flicks mean.

### JAWS

- **Modes.** A two finger rotate cycles three gesture modes: Touch Navigation, Text Reading and
  Speech Settings. The same flick means different things in each: in Touch Navigation a flick
  right moves to the next element, in Text Reading it moves to the next character, in Speech
  Settings it raises the speech rate.
- **Split tap.** With a finger held on an item, tapping anywhere with a second finger speaks the
  character at that location while exploring, or activates a key on the touch keyboard.
- **Rotor.** A two finger flick left or right cycles the element type used by the up and down
  flicks, the same rotor idea as VoiceOver.
- **Keyboard.** A four finger swipe down then up toggles the Windows touch keyboard, and while
  exploring it plays sounds as the finger enters and leaves the keyboard area.
- **Power gestures.** Five finger tap opens the JAWS manager, five finger double tap opens the
  JAWS window, five finger rotate turns JAWS gestures on and off.
- JAWS documents that a screen with five or more touch points is needed to use all of it.

## Android: the framework does most of the work

This is the model that matters most for Linux, because Android deliberately splits the work
between the system and the reader, and Linux has the same problem to solve.

- A service asks for touch responsibility by setting `FLAG_REQUEST_TOUCH_EXPLORATION_MODE`, which
  also requires the manifest attribute `canRequestTouchExplorationMode`.
- It then gets a `TouchInteractionController` per display, which receives all motion events for
  that display, and classifies each interaction by telling the framework to enter one of these
  states:
  - `STATE_TOUCH_INTERACTING`: the service is receiving the events and deciding.
  - `STATE_TOUCH_EXPLORING`: the user is exploring, so the framework converts the motion events
    into hover events, and applications then report what is under the finger. This is the
    exploration path, and the reader stops receiving motion events for the rest of that
    interaction.
  - `STATE_DRAGGING`: a two finger drag is passed through as a one finger gesture, which is how
    two finger scrolling works.
  - `STATE_DELEGATING`: the interaction is passed through untouched.
- Activation: `performClick()` first tries `ACTION_CLICK` on the item with accessibility focus,
  and falls back to simulating a click with motion events at the last location that had
  accessibility focus. `performLongClick()` does the same for a long press, and delegates
  movement afterwards, so double tap and hold can start a drag.
- The framework also detects standard gestures and delivers them by name through
  `onGesture`, with ids for tap, double tap, triple tap, swipes in each direction, two and three
  finger swipes, and tap and hold. A gesture event carries the motion events that produced it.
- Pass-through: `setTouchExplorationPassthroughRegion` and `setGestureDetectionPassthroughRegion`
  let a service name regions of the screen where touches bypass the reader entirely. This is how
  a reader can hand a game its raw touch.
- The framework also emits the events a reader watches: hover enter and exit, touch exploration
  gesture start and end, touch interaction start and end, gesture detection start and end.

### TalkBack and BackTalk

TalkBack is a service on top of that framework. BackTalk, the fork in use here, adds its own
layer: a two finger rotor, a circle menu, a status gesture, direct touch for audio games, sounds
with a matching vibration for every action, and braille keyboard dots that follow how the device
is held. The direct touch feature is the pass-through above with a friendlier name.

## iOS: VoiceOver is inside the operating system

VoiceOver is part of iOS, not a service layered on top, so it consumes touch directly and needs
no permission step. What is worth borrowing from it is the vocabulary and the conventions, which
have been copied widely since:

- Explore by touch: drag a finger over the screen and each item is spoken.
- Split tap: hold an item with one finger and tap elsewhere with a second finger, instead of
  selecting and double tapping.
- Scroll by page with a three finger swipe in the direction you want to move.
- The rotor: rotate two fingers like a dial to choose a setting, then flick up or down to use it.
- Touch typing on the keyboard in three modes, Standard, Touch and Direct Touch, where Direct
  Touch is the pass-through for a keyboard you already know.
- Double tap and hold, held until three rising tones, starts a drag.

## Side by side

| | Windows (NVDA, Narrator, JAWS) | Android (TalkBack, BackTalk) | iOS (VoiceOver) |
|---|---|---|---|
| Who grants touch | The program registers for pointer input | The service asks with a flag, the user consents | The system, no step needed |
| Who classifies gestures | The reader, from raw pointer messages | Mostly the framework, with raw events available | The system |
| Exploration | Hit test under the finger | Framework converts touches to hover events | Hit test under the finger |
| Activation | Accessibility action on the object | `ACTION_CLICK` first, then a simulated click | Accessibility activation |
| Pass-through | A toggle | Named screen regions | Direct Touch typing |
| Modes | A dedicated gesture, then per-mode meanings | BackTalk's rotor, reading controls | The rotor |

## Three ways to build it on Linux

### A. Portal-first, using what already exists

The pieces are all there:

- **Touch events**: the InputCapture portal, requested with the touchscreen capability, delivering
  touch through libei in receiver mode, with the permission prompt and remembered decision that
  portals already provide. This plays the part that `RegisterPointerInputTarget` plays on
  Windows.
- **The tree**: AT-SPI, or Newton or AT-SPI3 when they arrive, exactly as Orca uses it now.
- **Hit testing**: AT-SPI `GetAccessibleAtPoint(x, y, coord_type)`, which already exists per
  application.
- **Activation**: AT-SPI `AccessibleAction`, which is the same principle as Android's
  `performClick` trying `ACTION_CLICK` before simulating anything.
- **Gesture classification and modes**: written in the reader, using NVDA's vocabulary as the
  template, since that is the one proven on a desktop rather than a phone.

What has to be solved: coordinates. AT-SPI hit testing expects screen coordinates, and Wayland
has no global coordinate space; the portal delivers touch in the compositor's space, so mapping a
touch to a surface and then to tree coordinates is the open problem. Also, while capture is
active the application must not also act on the touch, which is the same problem Android solves
with its state machine.

### B. Compositor-side exploration, in the shape of Android's controller

Ask the compositor to do the hit testing, and give the reader results instead of raw touch. This
is what Android's `STATE_TOUCH_EXPLORING` does: the framework turns touches into hover events and
the application reports what is under the finger.

The natural home is the accessibility portal that is already proposed to replace the two
`org.freedesktop.a11y` interfaces. A touch explorer added there would fit the security argument
that proposal makes, because the reader never receives raw touch for the whole desktop and the
permission is a user decision. It also solves the coordinate problem, because the compositor
already knows which surface a touch belongs to.

What it needs: a design, then work in Mutter and KWin, then a reader that speaks it.

### C. Gestures only, as the smallest first step

Use InputCapture to give the reader gestures, not exploration: swipes and taps that run reader
commands, without reading what is under the finger. This is a fraction of the work of A, needs no
new specification, and can be tested with the portal as it ships today. Touch exploration could
then follow, once the coordinate and pass-through questions are settled.

For Orca, all three of these would land in the same place: a touch module alongside its existing
keyboard handling, taking its shape from NVDA's `touchHandler.py`, which is a single file holding
the classifier, the gesture definitions and the modes.

## What to measure first

These are the questions that decide whether A or C is possible at all, and they belong in
`experiments/`:

1. Does the InputCapture portal deliver touchscreen events to a plain client, on GNOME and on
   KDE?
2. What arrives: raw coordinates per touch, or libei gesture events, and do multiple fingers come
   as separate touch ids?
3. While capture is active, does the application underneath stop receiving touches?
4. Can capture be paused and resumed quickly enough for exploration, which needs to swallow some
   touches and let others through?

## Sources

- NVDA touch handling: https://github.com/nvaccess/nvda/blob/master/source/touchHandler.py
- NVDA hit testing: https://github.com/nvaccess/nvda/blob/master/source/NVDAObjects/__init__.py
- NVDA user guide, touch interaction: https://github.com/nvaccess/nvda/blob/master/user_docs/en/userGuide.md
- Narrator touch gestures, appendix B: https://support.microsoft.com/en-US/accessibility/windows/narrator/appendix-b-narrator-keyboard-commands-and-touch-gestures
- Narrator with touch, chapter 6: https://support.microsoft.com/en-us/accessibility/windows/narrator/chapter-6-using-narrator-with-touch
- JAWS touch support overview: https://freedomscientific.com/Content/Documents/Other/JAWSTouchSupportOverview.htm
- Android `TouchInteractionController`: https://developer.android.com/reference/android/accessibilityservice/TouchInteractionController
- Android `AccessibilityService`, gestures and motion events: https://developer.android.com/reference/android/accessibilityservice/AccessibilityService
- Android `AccessibilityGestureEvent`: https://developer.android.com/reference/kotlin/android/accessibilityservice/AccessibilityGestureEvent
- AT-SPI Component interface, `GetAccessibleAtPoint`: https://github.com/GNOME/at-spi2-core/blob/main/xml/Component.xml
- VoiceOver gestures on iPhone: https://support.apple.com/guide/iphone/use-voiceover-gestures-iph3e2e2281/ios
- The VoiceOver rotor: https://support.apple.com/en-us/111796
- BackTalk's differences from TalkBack: https://github.com/trypsynth/backtalk/blob/master/differences.md

## What was checked, and what was not

Checked directly, on 10 October 2026:

- NVDA's source, for the pointer message constants, the `RegisterPointerInputTarget` call, the
  `objectFromPoint` function, the hover actions and the gesture classes.
- NVDA's user guide, for the gesture vocabulary, the toggle and the touch modes.
- Narrator's gesture tables and its chapter on touch, and JAWS's touch support overview, both
  from their own documentation.
- Android's `TouchInteractionController`, `AccessibilityService` and `AccessibilityGestureEvent`
  documentation, including the state list, `performClick` and the pass-through regions.
- Apple's VoiceOver gestures and rotor documentation.
- AT-SPI's `Component.xml`, to confirm `GetAccessibleAtPoint`, `Contains` and `GetExtents` exist.

Not checked:

- No touch screen was used for any of this. Everything here is documentation and source reading.
- JAWS's internals are closed, so its mechanism is inferred from its documentation and from the
  fact that it runs on the same Windows pointer input. NVDA is the one whose mechanism was read
  directly.
- NVDA's `objectFromPoint` was confirmed to exist and to route to the platform APIs, but its
  internals were not read line by line.
- TalkBack's and BackTalk's source were not read for this document; the Android framework was,
  and BackTalk's own feature list is from its documentation.
