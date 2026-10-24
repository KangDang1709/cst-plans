# Teacher web (WebGV-CST)

Read the [Contract](contract.md) first, and use the names from [Terms](contract.md#terms). Designs: Figma "Trao đổi tech", Rung Chuông Vàng section (teacher frames).

## In short

Add one new screen, the **host screen**, where the teacher runs the game.
Everything before (create room, waiting room) and after (leaderboard) stays as it is.

## What to build

1. **Rename the mode value** (first task). In `examRooms.js`, keep `GOLDEN_BELL` but change its value from `'bell_alert'` to `'golden_bell'`. Search the repo for any other `bell_alert`.
2. **Turn the mode on.** Remove `disabled: true` from `GOLDEN_BELL`. The create form stays the same, including duration. (Do this last, when everything else works.)
3. **Route by mode.** When a golden_bell room starts, open the host screen (`GoldenBellHost`) instead of `/monitor`. `auto_advance` still goes to `/monitor`.
4. **Live data hook.** Add `useGoldenBellLive`, built on `useExamRoomRealtime`. It reads `state.liveQuestion`, and add a listener on `liveAnswers/{index}`. Use `useExamServerClock` for the countdown. On load and reconnect, call `GET /live`.
5. **Host screen**, following Figma:
   - Header: room name, number of students, time left in the room, **Kết thúc phòng thi**.
   - Question in the middle: reuse the image from `PreviewQuestionsPopup` (`screenshot_default_link` or `CanvasImage`), with the question countdown.
   - Left panel while answering: tabs **Đã trả lời / Chưa trả lời**, with counts.
   - Left panel after the question closes: tabs **Làm đúng / Làm sai**, plus the right/wrong % bar (from `GET .../results`).
   - **Hiện tên HS** toggle: show names, or avatars only.
   - Bottom: **Câu hỏi x/y** with a button that opens the question list (thumbnails, current one marked); **Câu trước**; **Bỏ qua thời gian** while answering; **Câu tiếp theo** after it closes; on the last question, **Tổng kết bài thi**, which goes to the leaderboard.
6. **Buttons call the API.** Câu trước, Câu tiếp theo and the question list all call `openQuestion(index)` (`POST .../open`). Bỏ qua thời gian calls `closeQuestion(index)` (`POST .../close`). Always send `expectedVersion`. On `409`, reload with `GET /live`.
7. **Dialogs.** Reuse the existing end and leave dialogs, with the texts from Figma ("Thời lượng cuộc thi vẫn còn…").

## Watch out for

- **Up to 1,000 students.** Only render the visible part of the student list and avatar grid (virtual list).
- **Disable buttons that would fail.** For example, Câu tiếp theo stays off while the question is answering.
- **Two tabs.** The second tab just follows Firebase. Version conflicts are handled by reloading.

## Done when

- A teacher can run a full game with a few students on real devices.
- Reloading the page in the middle of a question comes back to the same place.
- `auto_advance` rooms work exactly as before.
