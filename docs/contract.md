# Contract

This is what the backend, the teacher web and the student app must agree on.
If you change anything here, tell the other two teams.

## Question status

Each question in a room has one status:

| Status | Meaning |
| --- | --- |
| `NOT_STARTED` | Nobody has seen it yet |
| `ANSWERING` | It's on screen and the countdown is running |
| `DONE` | Time is up or the teacher skipped the time; results are ready |

```mermaid
stateDiagram-v2
    [*] --> NOT_STARTED
    NOT_STARTED --> ANSWERING: teacher opens it
    ANSWERING --> DONE: time runs out / teacher skips time / room ends
    DONE --> DONE: teacher opens it again (results only)
```

Only one question can be `ANSWERING` at a time. While it is, the teacher can't open another one.

## Firebase

Only the backend writes. The apps only read.

### `exam-rooms/{roomId}/state` (everyone listens, already exists)

We add one field, `liveQuestion`:

```json
{
  "state": "IN_PROGRESS",
  "startedAt": 1705315200000,
  "durationMinutes": 45,
  "timestamp": 1705315230000,
  "liveQuestion": {
    "index": 3,
    "status": "ANSWERING",
    "startedAt": 1705315230000,
    "deadlineAt": 1705315260000,
    "timeLimitSec": 30,
    "version": 7
  }
}
```

- `index` starts at 0.
- `liveQuestion` is missing until the teacher opens the first question.
- Count down using `deadlineAt` and the server clock (both apps already sync with the server clock). Never use the device clock alone.
- `version` goes up on every change. Ignore an update with a lower version than the one you have.

### `exam-rooms/{roomId}/liveAnswers/{index}/{studentId}` (teacher web only)

```json
{ "answeredAt": 1705315241000 }
```

One entry per student who pressed Nộp bài for that question. The teacher web uses it for the answered / not answered list. It does not say right or wrong.

## API

All routes sit next to the existing exam-room routes (same prefix and same login token).
Teacher calls that change the game send `expectedVersion`. If someone else changed the game first (a second tab or a double click), the backend answers `409 VERSION_CONFLICT`. Then reload with `GET /live`.

### Teacher

| Call | What it does |
| --- | --- |
| `GET /exam-rooms/{id}/live` | Current `liveQuestion`, the status of every question, and the server time. Use it on page load and reconnect. |
| `POST /exam-rooms/{id}/questions/{index}/open` | Body `{ expectedVersion }`. If the question is `NOT_STARTED`, starts it for everyone. If it's `DONE`, just shows it again (results only). Used by Câu trước, Câu tiếp theo and the question list. |
| `POST /exam-rooms/{id}/questions/{index}/close` | Body `{ expectedVersion }`. Ends the question now (Bỏ qua thời gian). Safe to call twice. |
| `GET /exam-rooms/{id}/questions/{index}/results` | Right / wrong counts and the list of students with their result. Only for `DONE` questions. |

`open` errors: `409 QUESTION_RUNNING` (another question is still `ANSWERING`) and `409 VERSION_CONFLICT`.

Results response:

```json
{
  "index": 3,
  "total": 30,
  "correct": 21,
  "wrong": 9,
  "students": [
    { "studentId": "s1", "name": "Nguyễn Văn Nam", "result": "CORRECT" },
    { "studentId": "s2", "name": "Lê Hồng Hạnh", "result": "WRONG" },
    { "studentId": "s3", "name": "Trần Minh", "result": "NO_ANSWER" }
  ]
}
```

`wrong` includes students who didn't answer (`NO_ANSWER`). `total` is everyone in the room.

### Student

| Call | What it does |
| --- | --- |
| `GET /exam-rooms/{id}/live` | Same as the teacher's, without other students' data. Use it on reconnect. |
| `POST /exam-rooms/{id}/questions/{index}/answer` | Sends the student's answer when they press Nộp bài. |

Answer request:

```json
{ "answer": { }, "isCorrect": true, "score": 1 }
```

- `answer` is the same answer data the app saves today for one question.
- The app checks the answer itself (like today) and sends `isCorrect`.

Answer errors:

| Code | When |
| --- | --- |
| `409 QUESTION_CLOSED` | The question isn't `ANSWERING`, or the time ended more than 1.5 s ago |
| `409 ALREADY_SUBMITTED` | The student already submitted this question |
| `403 NOT_IN_ROOM` | The student didn't join before the game started |

The 1.5 s grace covers an answer sent right before 0 on a slow network.

### Changes to existing calls

| Call | Change |
| --- | --- |
| `POST /join` | For a bell_alert room that has started, only students who already joined can get back in. Others get `403 GAME_STARTED`. |
| `POST /end` (and the timed end) | Ends any `ANSWERING` question first, then saves each student's total so the leaderboard works as today. |
| Create room | `mode = "bell_alert"` is now allowed. The duration stays and means "the whole room". |

## Scoring

- 1 point per right answer. No answer counts as wrong.
- Time used (for ties) = the sum of answer times across questions. Answer time = `answeredAt - question startedAt`. A question with no answer counts its full time limit.
- These fill the existing participant `Score`, `TotalScore` and `CompleteTime`, so the leaderboard needs no change.

## Settings (backend config)

| Name | Default | Meaning |
| --- | --- | --- |
| `BellAlert:QuestionTimeLimitSec` | 30 | Seconds per question until the PO decides otherwise |
| `BellAlert:MaxStudentsPerRoom` | 1000 | Join is refused above this |
| `BellAlert:AnswerGraceMs` | 1500 | Late answers accepted after the deadline |
