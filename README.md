# preview.html

A lightweight browser-side focus and visibility monitor designed for online quizzes.

The monitor detects changes in the quiz page's **visibility and browser-window focus** and sends structured events to the quiz application's backend.

It is intended to help administrators identify **possible interruptions or departures from the quiz page**, not to determine or accuse a student of cheating.

---

## What it detects

The monitor can report:

* `visibility_hidden` — the quiz page became hidden
* `visibility_visible` — the quiz page became visible again
* `window_blur` — the browser window lost focus
* `window_focus` — the browser window regained focus

Each event contains:

* Quiz/session identifier
* Sequential event number
* Event type
* Timestamp
* Current `document.hidden` state
* Current `document.hasFocus()` state

Example:

```json
{
  "sessionId": "student-session-12345",
  "sequence": 4,
  "type": "window_blur",
  "timestamp": "2026-09-25T13:50:21.123Z",
  "hidden": false,
  "focused": false
}
```

---

## How it works

The detector runs inside the student's browser.

```text
Student Browser
      │
      │ focus / visibility event
      ▼
Focus Monitor
      │
      │ structured event
      ▼
Quiz Backend
      │
      ▼
Administrator Dashboard
```

The monitor itself does not require access to the student's computer outside the browser.

The backend integration is intentionally left configurable because the quiz application may use its own API, WebSocket connection, authentication system, or other architecture.

---

## Split-screen behavior

The monitor can detect some focus changes when a student uses split-screen or multiple windows.

For example:

```text
┌──────────────────┬──────────────────┐
│                  │                  │
│   Quiz           │   Other window   │
│                  │                  │
└──────────────────┴──────────────────┘
```

If clicking the other window causes the quiz browser window to lose focus, a `window_blur` event may be generated.

However, browsers do **not** provide a reliable API that allows a normal webpage to determine exactly what the user is looking at.

Therefore, the monitor cannot reliably determine:

* What is displayed in another window
* What is displayed in another application
* Whether the student is physically looking at another screen area
* Whether a focus change was intentional
* Whether a focus/visibility event represents cheating

---

# Important Disclaimer

## Focus events are NOT proof of cheating

A focus or visibility event should be treated only as a **technical signal**.

For example, a `window_blur` event may occur because of:

* Clicking another window
* An operating-system dialog
* Browser UI interaction
* Another application requesting focus
* Accidental interaction
* Other browser or operating-system behavior

Likewise, browser behavior can vary between operating systems, browsers, devices, and window configurations.

**The system must not automatically interpret a focus event as proof that a student cheated.**

Any disciplinary or academic decision should be made by an authorized human using appropriate context and the applicable school rules.

---

## Privacy considerations

The monitor is intentionally limited to browser focus and visibility information.

It does **not**:

* Capture screenshots
* Record the student's screen
* Record the webcam
* Record microphone audio
* Read the contents of other browser tabs
* Read other applications
* Determine what the student is viewing
* Provide unrestricted access to the student's computer

The quiz application should collect only the information necessary for its stated purpose.

If focus events are stored on a server, the quiz application should have an appropriate privacy notice explaining what is collected, why it is collected, how long it is retained, and who can access it.

Access to stored student/session information should also be appropriately restricted.

---

## Backend integration

The supplied detector intentionally does not assume a specific backend endpoint.

The application developer should connect:

```js
sendEvent(event)
```

to the quiz application's existing server infrastructure.

Possible implementations include:

* HTTP `fetch()` requests
* WebSockets
* An existing quiz API
* Another server-side event system

The backend should validate incoming events rather than blindly trusting data supplied by the browser.

A browser client can be modified by its user, so client-generated events should not be treated as cryptographically guaranteed evidence.

---

## Recommended administrator display

Instead of displaying:

> CHEATING DETECTED

use neutral language such as:

> Focus signal detected

or:

> Quiz page lost browser focus

Example:

```text
Student: 12345
Quiz: Mathematics Quiz

Status: Active

Focus events
────────────────────────────
14:02:11  Window lost focus
14:02:14  Window regained focus
14:07:32  Page became hidden
14:07:38  Page became visible
```

This preserves the distinction between an observable browser event and an interpretation of that event.

---

## Browser limitations

This project operates within normal browser security restrictions.

It cannot guarantee detection of every situation in which a student may leave the quiz.

It also cannot guarantee that every detected event represents intentional activity.

Browser APIs and behavior may differ across:

* Chrome
* Edge
* Firefox
* Safari
* Mobile browsers
* Different operating systems
* Different window-management configurations

Testing should therefore be performed on the actual devices and browsers used for the quiz.

---

## Intended use

This project is intended as a **supplementary quiz-monitoring signal** for educational environments.

It should not be used as the sole basis for accusing a student of misconduct or making an academic penalty.

The final interpretation of events should remain with the teacher or other authorized staff.

---

## License / ownership

This README does not establish ownership or licensing of the surrounding quiz application.

The quiz application's developer/owner should add the appropriate license, copyright notice, and privacy documentation for their project.
