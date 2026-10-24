# Teacher web (WebGV-CST)

Read the [Contract](contract.md) first. Designs: Figma "Trao đổi tech", Rung Chuông Vàng section (teacher frames).

## In short

Add one new screen, the **host screen**, where the teacher runs the game.
Everything before (create room, waiting room) and after (leaderboard) stays as it is.

## What to build

1. **Turn the mode on.** In `examRooms.js`, remove `disabled: true` from `GOLDEN_BELL`. The create form stays the same, including duration. (Do this last, when everything else works.)
2. **Route by mode.** When a bell_alert room starts, open the host screen instead of `/monitor`. `auto_advance` still goes to `/monitor`.
3. **Live data hook.** Extend `useExamRoomRealtime` to also read `state.liveQuestion`, and add a listener on `liveAnswers/{index}`. Use `useExamServerClock` for the countdown. On load and reconnect, call `GET /live`.
4. **Host screen**, following Figma:
   - Header: room name, number of students, time left in the room, **Kết thúc phòng thi**.
   - Question in the middle: reuse the image from `PreviewQuestionsPopup` (`screenshot_default_link` or `CanvasImage`), with the question countdown.
   - Left panel while answering: tabs **Đã trả lời / Chưa trả lời**, with counts.
   - Left panel after the question ends: tabs **Làm đúng / Làm sai**, plus the right/wrong % bar (from `GET .../results`).
   - **Hiện tên HS** toggle: show names, or avatars only.
   - Bottom: **Câu hỏi x/y** with a button that opens the question list (thumbnails, current one marked); **Câu trước**; **Bỏ qua thời gian** while answering; **Câu tiếp theo** after it ends; on the last question, **Tổng kết bài thi**, which goes to the leaderboard.
5. **Buttons call the API.** Câu trước, Câu tiếp theo and the question list all call `open` with the question number. Bỏ qua thời gian calls `close`. Always send `expectedVersion`. On `409`, reload with `GET /live`.
6. **Dialogs.** Reuse the existing end and leave dialogs, with the texts from Figma ("Thời lượng cuộc thi vẫn còn…").

## Watch out for

- **Up to 1,000 students.** Only render the visible part of the student list and avatar grid (virtual list).
- **Disable buttons that would fail.** For example, Câu tiếp theo stays off while the question is answering.
- **Two tabs.** The second tab just follows Firebase. Version conflicts are handled by reloading.

## Done when

- A teacher can run a full game with a few students on real devices.
- Reloading the page in the middle of a question comes back to the same place.
- `auto_advance` rooms work exactly as before.
