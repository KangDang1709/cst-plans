# Backend (pcnl-svc-exam-net)

Read the [Contract](contract.md) first. This page says how to build it.

## In short

Add a **GoldenBell** part to the existing exam service. Don't make a new service: rooms, joining, start/end and results are already here.
Use the names from [Terms](contract.md#terms) for every class, route and field.
Follow the existing style: MediatR commands, DynamoDB, and `FirebaseService` to write to Firebase.

## What to store (DynamoDB)

| Item | Key | Fields |
| --- | --- | --- |
| Live state | `ROOM#{roomId}` / `LIVE` | `currentIndex`, `version` |
| Question | `ROOM#{roomId}` / `Q#{index}` | `status`, `startedAt`, `deadlineAt`, `timeLimitSec`, and after it ends: `correct`, `wrong`, `total` |
| Answer | `ROOM#{roomId}#Q#{index}` / `STUDENT#{studentId}` | `answer`, `isCorrect`, `score`, `answeredAt` |

- Change the live state and question items with a check on `version` (conditional write). If the check fails, return `409 VERSION_CONFLICT`.
- Save an answer only if none exists yet (`attribute_not_exists`). Otherwise return `409 ALREADY_SUBMITTED`.
- Whether these go in the existing table or a new one: check the current key design and choose. Note the choice here.

## What to build

Each ticket below is one Trello card and one PR. Do them **top to bottom**. If a ticket needs a part from another team that isn't ready yet, fake it with the [Contract](contract.md): mock the API response, or type `liveQuestion` into Firebase by hand.

### BE-1 · Mode, live state and opening questions (about 5 days)

Needs: nothing. **Do this first: web and Unity build on it.**

- Rename `bell_alert` to `golden_bell` everywhere, starting with the comment on `Mode` in `src/Domain/Entities/ExamRoom.cs`. Add a constant for it. Accept `mode = "golden_bell"` when creating a room.
- Add the config values `GoldenBell:*` from the [Contract](contract.md#settings-backend-config).
- Create the DynamoDB items above.
- `GetLiveState` (`GET .../live`): the live state, every question's status and the server time.
- `OpenQuestion` (`POST .../open`):
  - If another question is `ANSWERING`, return `409 QUESTION_RUNNING`.
  - If this one is `NOT_STARTED`: set it to `ANSWERING` with `startedAt` and `deadlineAt`.
  - If it's `DONE`: only move `currentIndex` to it.
  - Write `liveQuestion` to Firebase.
- Join rule: once a golden_bell room has started, only students who joined before can get back in. Refuse joins above `MaxStudentsPerRoom`.

Done when: a test can create a golden_bell room, start it, open question 0 and see `liveQuestion` in Firebase. A second `open` while it's answering gets `409`. Unit tests cover these.

### BE-2 · Answers, closing and results (about 5 days)

Needs: BE-1.

- `SubmitAnswer` (`POST .../answer`): check the student is in the room, the question is `ANSWERING` and the time (plus 1.5 s) hasn't passed. Save the answer, then write `liveAnswers/{index}/{studentId}` to Firebase.
- `CloseQuestion` (`POST .../close`), safe to call twice: if it's `ANSWERING`, count the answers, save `correct`, `wrong` and `total`, set it to `DONE`, and write `liveQuestion` to Firebase.
- Timed close: when a question opens, send an SQS message delayed by the time limit. Its handler calls `CloseQuestion`. (The EventBridge Scheduler we use for room start/end is too slow for 30-second questions.)
- `GetQuestionResults` (`GET .../results`): the saved counts plus each student's result, with `NO_ANSWER` for students who didn't submit.

Done when: a question closes by itself when time runs out, even with no teacher tab open. Results are right for students who answered right, wrong and not at all. Late, double and not-in-room answers get the right errors. Unit tests cover these.

### BE-3 · End and leaderboard (about 2 days)

Needs: BE-2.

- In the end command (both the teacher's end and the timed end from `StartExamCommand`), for golden_bell rooms: close the running question, then add up each student's points and time into `Score`, `TotalScore` and `CompleteTime` ([Scoring](contract.md#scoring)).

Done when: after a game, the existing leaderboard shows the right ranking without any change to it.

### BE-4 · Load test and test with web and Unity (about 4 days)

Needs: BE-3.

- Load test: 1,000 fake students answer one question in the last few seconds. Fix what breaks.
- Play full games with the web and Unity devs and fix the bugs found.

Done when: the 1,000-student test passes with no errors, and a full game on real devices works end to end.

## Watch out for

- **1,000 students in a room.** Don't keep one counter that every answer updates. Count when the question closes instead. Results queries may need paging.
- **Firebase writes.** Use the existing REST PATCH and add `liveQuestion` without wiping the other `state` fields.
- **Don't break auto_advance.** Branch on `room.Mode` and leave the current flow alone.
