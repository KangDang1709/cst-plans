# Backend (pcnl-svc-exam-net)

Read the [Contract](contract.md) first. This page says how to build it.

## In short

Add a **BellAlert** part to the existing exam service. Don't make a new service: rooms, joining, start/end and results are already here.
Follow the existing style: MediatR commands, DynamoDB, and `FirebaseService` to write to Firebase.

## What to store (DynamoDB)

| Item | Key | Fields |
| --- | --- | --- |
| Live game | `ROOM#{roomId}` / `LIVE` | `currentIndex`, `version` |
| Question | `ROOM#{roomId}` / `Q#{index}` | `status`, `startedAt`, `deadlineAt`, `timeLimitSec`, and after it ends: `correct`, `wrong`, `total` |
| Answer | `ROOM#{roomId}#Q#{index}` / `STUDENT#{studentId}` | `answer`, `isCorrect`, `score`, `answeredAt` |

- Change the live game and question items with a check on `version` (conditional write). If the check fails, return `409 VERSION_CONFLICT`.
- Save an answer only if none exists yet (`attribute_not_exists`). Otherwise return `409 ALREADY_SUBMITTED`.
- Whether these go in the existing table or a new one: check the current key design and choose. Note the choice here.

## What to build

1. **Allow the mode.** Accept `mode = "bell_alert"` when creating a room.
2. **Join rule.** In join: if the room is bell_alert and already started, only let back in students who joined before. Also refuse joins above `MaxStudentsPerRoom`.
3. **Open a question.** Command `OpenQuestion`:
   - If another question is `ANSWERING`, return `409 QUESTION_RUNNING`.
   - If this one is `NOT_STARTED`: set it to `ANSWERING`, set `startedAt` and `deadlineAt`, and schedule the timed close (below).
   - If it's `DONE`: only move `currentIndex` to it.
   - Write `liveQuestion` to Firebase.
4. **Close a question.** Command `CloseQuestion`, safe to call twice:
   - If it's not `ANSWERING`, do nothing.
   - Count the answers (one query on the answer items), save `correct`, `wrong` and `total`, and set it to `DONE`.
   - Write `liveQuestion` to Firebase.
5. **Timed close.** When a question opens, send an SQS message with a delay equal to the time limit. The handler calls `CloseQuestion`. This works even if the teacher's tab is closed.
   The EventBridge Scheduler we use for room start/end is too slow for 30-second questions, so don't use it here.
6. **Submit an answer.** Check the student is in the room, the question is `ANSWERING`, and the time (plus 1.5 s) hasn't passed. Save the answer, then write `liveAnswers/{index}/{studentId}` to Firebase.
7. **Results.** `GET .../results` reads the saved counts and the answer items, and adds the students with no answer as `NO_ANSWER`.
8. **Live state.** `GET .../live` returns the live game, every question's status and the server time.
9. **End.** In the end command (teacher end and the timed end from `StartExamCommand`), for bell_alert rooms: close the running question, then add up each student's points and time into `Score`, `TotalScore` and `CompleteTime`. The existing leaderboard then works as is.

## Watch out for

- **1,000 students in a room.** Don't keep one counter that every answer updates. Count when the question closes instead. Results queries may need paging.
- **Firebase writes.** Use the existing REST PATCH and add `liveQuestion` without wiping the other `state` fields.
- **Don't break auto_advance.** Branch on `room.Mode` and leave the current flow alone.

## Done when

- Unit tests cover open, close (including twice), submit (late, twice, not in room) and end.
- A load test with 1,000 fake students answering one question passes without errors.
