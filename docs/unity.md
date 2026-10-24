# Student app (ConSangTao)

Read the [Contract](contract.md) first. Designs: Figma "Trao đổi tech", Rung Chuông Vàng section (student frames).

## In short

The student doesn't move between questions anymore. **The app shows whatever question the teacher has opened**, and the student answers it before the time runs out.
We keep the existing question screen and add a new controller that follows the teacher.

## What to build

1. **Route by mode.** In `ExamLobbyController`, read `ExamRoomDetailData.mode`. For `bell_alert`, open the question scene with the new controller. `auto_advance` stays as it is.
2. **New controller.** `BellAlertQuestionSceneController`, based on `ExamQuestionSceneController`:
   - Listen to `exam-rooms/{id}/state` (already done in the lobby) and read `liveQuestion`.
   - When `status` is `ANSWERING`, go to that question with `EnterQuestionByIndex(index)`.
   - Hide the student's own question navigation and the whole-exam timer.
3. **Countdown.** Show seconds left until `deadlineAt`, using `ExamRoomTimingSync` (server clock). At 0, lock the answer.
4. **Submit.** Use the existing in-game menu (Thoát, Tắt âm thanh, Làm lại, **Nộp bài**). On Nộp bài:
   - Check the answer like today (`CheckCorrectedAnswers`).
   - Send it with `POST .../questions/{index}/answer`.
   - Lock the question. A student submits once per question.
5. **After the question ends** (`status` becomes `DONE`):
   - Show **Bạn đã trả lời ĐÚNG!** or **Bạn đã trả lời SAI...** (Figma). No answer counts as wrong.
   - Then show **Vui lòng chờ một chút nhé, câu tiếp theo sắp đến rồi!** until the next question opens.
   - Don't show the correct answer for now.
6. **Teacher goes back to an ended question.** Keep showing the wait screen.
7. **Reconnect.** When the app comes back (network or resume), call `GET /live` and jump to the current state.
8. **End.** When `state` becomes `ENDED`, show the end screen like today.

## Watch out for

- **Speed.** Today, leaving a question uploads a screenshot, and the final submit reloads every question. Skip both in bell_alert, or the app will fall behind the teacher.
- **Loading a question takes about 1.5 s.** Start the countdown from `deadlineAt`, not from when loading finishes, so all students end at the same time.
- **Don't break auto_advance.** All new behaviour sits in the new controller.

## Done when

- Several phones follow the teacher through a full game and stay in sync.
- Turning Wi-Fi off and on in the middle of a question puts the app back on the right question.
- Answering at the last second is accepted, and answering after the time is up is refused politely.
