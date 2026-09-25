# tab-focus-demo

## Implementation Guide

This document explains how to integrate the new `QuizFocusMonitor` into the existing quiz application.

The reference/demo implementation is:

`hongyuekai427-bit/tab-focus-demo/preview.html`

The current demo is a standalone browser focus detector. It displays `document.hidden`, `document.hasFocus()`, window focus state, fullscreen state, and a local event log. The new implementation turns the detector into a reusable module that can report structured events to the quiz application's existing backend.

---

# 1. Integration goal

The final system should work like this:

```text
Student's browser
        │
        │ browser focus/visibility event
        ▼
QuizFocusMonitor
        │
        │ structured event
        ▼
Existing quiz frontend
        │
        │ existing API / WebSocket
        ▼
Existing quiz backend
        │
        ▼
Teacher/admin interface
```

Do not create an unrelated second quiz system.

Do not replace the existing quiz architecture.

Integrate the monitor into the existing application.

---

# 2. Existing demo

The repository's current `preview.html` already contains the basic detection mechanisms:

* `document.visibilitychange`
* `window.blur`
* `window.focus`
* `document.fullscreenchange`
* `document.hidden`
* `document.hasFocus()`

The current implementation writes events only to the page's local log.

It does not currently send those events to a server.

The new implementation should preserve the useful detection behavior while separating detection from backend communication.

---

# 3. Add the new monitor

Add the supplied `QuizFocusMonitor` implementation to the quiz frontend.

Prefer placing it in its own JavaScript module if the application already uses modules.

For example:

```text
src/
├── ...
├── quiz-focus-monitor.js
└── ...
```

If the existing application is a simple HTML/JavaScript application, the monitor may instead be included directly in the page.

Do not restructure the entire application just to add this feature.

---

# 4. Initialize the monitor

The monitor needs the quiz application's existing session identifier.

Example:

```js
const focusMonitor = new QuizFocusMonitor({
    sessionId: currentQuizSessionId
});

focusMonitor.start();
```

`currentQuizSessionId` is only an example.

Use the application's actual authenticated quiz/session identifier.

Do not generate a new unrelated identity if the existing application already has one.

---

# 5. Event types

The monitor produces four main event types.

```text
visibility_hidden
visibility_visible
window_blur
window_focus
```

Example event:

```json
{
  "sessionId": "student-session-12345",
  "sequence": 4,
  "type": "window_blur",
  "timestamp": "2026-09-25T13:50:21.123Z",
  "browserState": {
    "hidden": false,
    "focused": false
  }
}
```

The exact session ID and authentication information must come from the existing quiz system.

---

# 6. Backend connection

This is the most important integration point.

The supplied monitor contains:

```js
async sendEvent(event) {
    // backend connection goes here
}
```

The developer implementing the quiz should replace this method with the application's existing server communication mechanism.

## If the application uses HTTP

For example:

```js
async sendEvent(event) {
    try {
        await fetch("/EXISTING/QUIZ/EVENT/ENDPOINT", {
            method: "POST",
            headers: {
                "Content-Type": "application/json"
            },
            credentials: "include",
            body: JSON.stringify(event)
        });
    } catch (error) {
        console.error(
            "[QuizFocusMonitor] Failed to send event:",
            error
        );
    }
}
```

`/EXISTING/QUIZ/EVENT/ENDPOINT` is intentionally a placeholder.

Do not use that literal path unless the application's backend actually provides it.

The developer must identify the existing endpoint from the quiz application's backend.

---

# 7. If the application uses WebSockets

If the existing quiz already maintains a WebSocket connection, reuse it.

Example:

```js
sendEvent(event) {
    if (
        quizSocket &&
        quizSocket.readyState === WebSocket.OPEN
    ) {
        quizSocket.send(JSON.stringify({
            type: "focus_event",
            data: event
        }));
    }
}
```

Do not create a second WebSocket connection if the application already has an appropriate authenticated connection.

---

# 8. If the application has no backend endpoint yet

The frontend developer should coordinate with the backend developer.

The backend needs a way to accept focus-monitor events and associate them with the authenticated quiz session.

The backend should validate:

* Authentication
* Quiz/session identity
* Event structure
* Event type
* Timestamp format
* Sequence number

Do not blindly trust arbitrary student-supplied session IDs.

The browser is an untrusted client.

---

# 9. Recommended backend event structure

The backend can store something equivalent to:

```json
{
  "sessionId": "...",
  "sequence": 12,
  "type": "visibility_hidden",
  "timestamp": "...",
  "browserState": {
    "hidden": true,
    "focused": false
  }
}
```

Additional server-side metadata may be added by the existing application if appropriate.

Do not collect unnecessary information.

---

# 10. Admin dashboard

The administrator interface should present the events as neutral technical signals.

Recommended:

```text
Student: 12345

Focus events
────────────────────────────
14:02:11  Window lost focus
14:02:14  Window regained focus
14:07:32  Page became hidden
14:07:38  Page became visible
```

Avoid wording such as:

```text
CHEATING DETECTED
STUDENT CAUGHT
STUDENT LEFT TO CHEAT
```

A focus event does not establish why the event happened.

The teacher/admin can use the event history together with the quiz context and school procedures.

---

# 11. Do not treat every event as separate misconduct

`blur` and `visibilitychange` may happen close together during the same user action.

The supplied monitor already includes basic duplicate suppression.

The backend/admin UI should also avoid making the interface misleading by presenting two simultaneous browser events as two completely separate incidents.

If desired, the backend can group events occurring within a short time window into one focus transition.

Do not invent a punitive threshold without the teacher's requirements.

---

# 12. Split-screen

The monitor can detect some split-screen interactions.

Example:

```text
┌──────────────────┬──────────────────┐
│                  │                  │
│       QUIZ       │   OTHER WINDOW   │
│                  │                  │
└──────────────────┴──────────────────┘
```

If the quiz window loses browser focus, the monitor can report:

```text
window_blur
```

When the quiz regains focus:

```text
window_focus
```

However, a normal webpage cannot reliably determine what is displayed in another window or whether the student is looking at another part of the screen.

Do not attempt to bypass browser security restrictions to obtain that information.

---

# 13. Visibility versus focus

These signals are different.

### `document.hidden`

Indicates whether the browser considers the document hidden.

### `document.hasFocus()`

Indicates whether the document currently has focus.

### `blur`

Reports that the window lost focus.

### `focus`

Reports that the window regained focus.

Do not collapse all four signals into a claim such as "student switched tabs."

Instead, preserve the actual event type.

---

# 14. Fullscreen

The existing demo also monitors:

```js
fullscreenchange
```

Fullscreen monitoring can remain in the demo if useful.

It should not automatically be interpreted as a focus event.

If the quiz requires fullscreen, implement the application's existing fullscreen policy separately.

Do not assume fullscreen prevents every possible way of leaving the quiz.

---

# 15. Lifecycle

Start the monitor when the actual quiz session begins:

```js
focusMonitor.start();
```

Stop it when the quiz session legitimately ends:

```js
focusMonitor.stop();
```

For example:

```text
Quiz starts
    ↓
focusMonitor.start()
    ↓
Student answers questions
    ↓
focus/visibility events recorded
    ↓
Student submits quiz
    ↓
focusMonitor.stop()
```

Do not leave the monitor running on unrelated pages of the application unless that is explicitly required.

---

# 16. Reliability

The frontend should not crash if the backend is temporarily unavailable.

For example:

```js
async sendEvent(event) {
    try {
        await fetch(...);
    } catch (error) {
        console.error(
            "[QuizFocusMonitor] Event delivery failed:",
            error
        );
    }
}
```

If reliable event delivery is important, the backend/frontend team may implement a small retry or queue mechanism.

Any retry system should avoid producing duplicate server records.

---

# 17. Security

The browser is not a trusted environment.

Students can modify client-side JavaScript, disable JavaScript, alter requests, or use a different browser environment.

Therefore:

* Do not treat browser events as cryptographically guaranteed evidence.
* Authenticate the quiz session on the server.
* Validate incoming event data.
* Do not trust a client-provided student identity by itself.
* Do not expose administrative event data to students.
* Do not put backend secrets/API keys in frontend JavaScript.
* Use the application's existing authentication system.

---

# 18. Privacy

Only collect the information necessary for the quiz-monitoring purpose.

This detector does not need to:

* Capture screenshots
* Record the webcam
* Record microphone audio
* Read other browser tabs
* Read other applications
* Inspect another window
* Record the student's screen

If events are stored, the quiz application should document:

* What information is collected
* Why it is collected
* Who can access it
* How long it is retained
* How it is protected

Follow the school's applicable privacy requirements.

---

# 19. Disclaimer

Focus and visibility events are **technical browser signals, not proof of cheating**.

A student may lose focus because of:

* An operating-system dialog
* Browser UI
* An accidental click
* Another legitimate application
* Notifications
* Window-management behavior
* Browser-specific behavior
* Device-specific behavior

Different browsers and operating systems may also report focus and visibility differently.

The system therefore must not automatically convert a focus event into an accusation of misconduct.

Any academic decision should remain with the authorized teacher/school staff and follow the applicable quiz and school rules.

---

# 20. Testing checklist

Before deployment, test at minimum:

* [ ] Normal tab switch
* [ ] Return to quiz tab
* [ ] Alt+Tab
* [ ] Minimize browser
* [ ] Restore browser
* [ ] Split-screen
* [ ] Clicking another window
* [ ] Clicking back into quiz
* [ ] Fullscreen enter
* [ ] Fullscreen exit
* [ ] Browser refresh
* [ ] Quiz submission
* [ ] Network temporarily unavailable
* [ ] Multiple simultaneous browser events
* [ ] Session ending
* [ ] Multiple students/sessions simultaneously

Test using the same browsers and devices that students will actually use.

---

# 21. Integration acceptance criteria

The implementation is complete when:

1. The existing quiz functionality still works.
2. Focus monitoring starts only for an active quiz session.
3. Focus/visibility events are generated correctly.
4. Events contain the correct authenticated quiz/session association.
5. Events reach the existing backend.
6. The admin can view the recorded events.
7. Duplicate browser events are handled appropriately.
8. Temporary network failures do not crash the quiz.
9. No backend secrets are exposed to the browser.
10. The system does not describe a focus event as definitive proof of cheating.
11. No unnecessary screen, camera, microphone, or cross-application data is collected.
12. Existing quiz functionality is not removed or unnecessarily rewritten.

---

# 22. Important instruction for the implementing developer

**Inspect the existing quiz frontend and backend before modifying anything.**

Determine:

* How quiz sessions are identified
* How students are authenticated
* How the frontend communicates with the backend
* Whether an existing WebSocket/API can be reused
* Where quiz lifecycle events are handled
* Where admin events are displayed
* How server-side data is stored

Then integrate `QuizFocusMonitor` into those existing systems.

Do not invent an API endpoint, database schema, authentication mechanism, or WebSocket protocol when an existing implementation is already available.

Preserve unrelated functionality.

Make the smallest change necessary to add focus/visibility monitoring.
