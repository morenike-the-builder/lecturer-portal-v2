# Spec: lecturer portal flow improvements (v1)

Prototype: `index.html`. The "Demo" and "Reset" chips at the bottom of the dashboard replay each state.

Owner: Samson implements. Product owns the flow and design.

Four improvements:
1. New-course setup checklist
2. Material-request alerts
3. Add course (single flow, multiple courses)
4. End semester (lecturer-led) and the prompts around it

Principle for v1: keep it simple. Lecturers learn to add courses themselves. Nothing is ever ended or archived automatically.

---

## 1. New-course setup checklist

**Problem:** adding a course to an account that already has courses replaced the dashboard with setup prompts.

**When it appears:** only after a course is added (never on a fresh dashboard), for an account that already has at least one course. An account with zero courses still gets the full first-time onboarding.

**Where it shows (one source of truth, three places):**
- **Dashboard:** a card under the welcome row, "Complete setup for BIO 201". It has a progress bar and 3 numbered steps (Add course outline · Schedule a class · Upload resources). The next step is the primary button. All existing dashboard summaries stay below, unchanged.
- **Courses page:** the new course appears in the course list tagged "New · Setup needed", with "n of 3 done", a progress bar and a **Next: …** button.
- **Course page:** a setup card at the top for that course.

**Multiple new courses:** one card, one block per course. At most 3 are shown, with a "+N more" note.

**Step completion:** driven by data, not clicks: at least one outline topic, one scheduled class, one resource on the course. Completing a step anywhere ticks it.

**Disappears:**
- Dashboard: when all 3 steps are done ("BIO 201 is ready" for about 4 seconds, then removed), or when the lecturer taps ×. Hiding only hides the dashboard card. The Courses page and the course page keep showing the setup status. It returns on next sign-in if steps are incomplete.
- Courses page: the card stays in the list permanently and shows "✓ Setup complete" once done.

---

## 2. Material-request alerts

**Trigger:** a student opens a course with no materials and taps Request materials. As today, the request goes to Content Operations. It now also goes to the lecturer(s) linked to that course.

**Dashboard:** the first item in the **Action Needed** card:
- Title: "Students are requesting materials for BIO 201".
- Body: "Upload a resource to help them begin studying."
- Button: **Upload materials**, which opens that course's Resources tab with the upload dialog open.
- No counts or dates on the item (kept clean on purpose).

**Email:**
- Subject: "Students are requesting materials for BIO 201".
- Body: "Hi Samson, N students opened BIO 201 · Introduction to Cell Biology and asked for study materials. Upload a resource to help them begin studying. PDFs, slides and YouTube links are visible to students immediately."
- Button: **Upload materials**.
- Footer: the frequency rule below.

**Grouping (no email per student):**
- One alert and one email thread per course. Repeat requests from the same student are counted once.
- The first request sends an email immediately.
- Further requests only update the count. They are rolled into a digest of at most 1 email per course per day.
- If still unaddressed, a reminder after 3 days, up to 2 reminders. After that, only the dashboard alert remains.
- Co-taught courses: every linked lecturer is notified. One upload resolves it for all.

**No lecturer linked:**
- The request goes to Content Operations only, flagged "No lecturer linked".
- Notify the department admin or coordinator as a fallback (to confirm with Ops).
- Requests keep accumulating. When a lecturer is linked or claims the course, they see the open alert with the full count.

**Addressed when:**
- A resource is uploaded to that course for the requesting session and semester.
- The dashboard item shows "✓ Request for BIO 201 addressed" for a few seconds, then clears.
- The requesting students get "Materials are now available".
- Content Ops' request closes with it.
- Requests can only be made while a course has no materials, so no new request opens after an upload.

**Backend notes:**
- Match request → course → linked lecturer(s) by course ID and session.
- Store one `material_request` group per course and session: `student_count`, `first_requested_at`, `last_emailed_at`, `reminders_sent`, `status` (open/addressed).
- Resolve on resource-created for that course. Don't resolve on edits or deletes.
- This is separate from the missing-course request (a course doesn't exist), which stays Content Ops only.

---

## 3. Add course

**Problem:** existing-course selection and manual entry were shown together, which was confusing.

**Two paths, never shown together:**
1. **Pick existing courses (default):**
   - Courses taught last semester (just archived) are listed first, tagged "carry over".
   - A searchable list of department courses with checkboxes. The lecturer can tick several.
   - Ticked courses show in one Review box (title, code, units, department, year, semester). Nothing to type.
   - Academic year and semester selectors, and an optional date: "When does this semester end?", applied to all ticked courses.
   - One confirm: "Add course" or "Add N courses".
   - Courses the lecturer already teaches are greyed out ("Already in your courses"). An archived course becomes available again.
2. **Enter new record:** only after tapping "Can't find your course? **Enter new record**".
   - Required: course title, code, units, academic year, department, semester. Optional: description and the end date.
   - One course at a time. After saving: "Add another new record", "Pick from course list" or "Done".

**Acceptance criteria:**
- The manual form is hidden in the initial view.
- Selecting an existing course shows its details without asking for them again.
- Manual fields appear only after Enter new record.
- Both complete flows are never displayed at once.

**Edit Course** (course page) uses the same dialog, prefilled, with Save changes. It has no course list and no back link.

**Every added course** gets its own setup checklist (section 1).

---

## 4. End semester (lecturer-led)

**Principles:**
- Never automatic. The lecturer ends the semester themselves, because ungraded work may remain.
- Terminology: "Session Management" is retired. Use **End semester**, **Teach this course again** and **Past semesters**.
- Archived semesters are view-only under Past semesters. No restore.

**Where:**
- Courses page: **End semester** (all current courses).
- Course page: **End semester** (that course).
- Dashboard prompts (below).

**Dialog: one screen, then a short confirmation.**
1. **One screen:** "End 1st Semester 2025/2026?"
   - One line: "Your students, attendance and grades are saved under Past semesters. Nothing is deleted."
   - The courses, one line each. On the Courses page each has a checkbox ("Untick any course that is still running"). Started from a course page, it shows that course only.
   - Each course shows "✓ All done", or "⚠ N things left ▾". The unfinished items (syllabus topics not ticked off, submissions not graded, scheduled announcements) are hidden until the lecturer expands them, and each has a link (Syllabus tracker, Grade, Review).
   - One primary button, **End semester**, and **Cancel**. Warnings never block.
2. **Confirmation:** "CSC 401, CSC 305 archived. Find them any time under Past semesters." with **Add next semester's courses** and **Not now**.

**Teaching a course again** is not a separate flow. Courses just archived appear first in the add-course list, tagged "Taught last semester · outline & resources carry over". The lecturer ticks the ones they are teaching again, chooses the academic year and semester (it defaults to the 2nd semester), and adds them with everything else. Carried-over courses only need "Schedule a class" in their setup checklist.

**Events recorded** (what the follow-up emails read): one `semester_ended` record per course: `course`, `semester`, `program`, `expectedEnd` (from add-course, may be empty), `endedOn` (the day the lecturer pressed End semester), `recordedAt`, `source: lecturer`.

**Dashboard prompts (neither ends anything):**
- **Semester ending:** shown from 3 days before a course's expected end date until it is ended. "Your semester ends in 2 days (30 Oct). When you're ready, end the semester… Nothing happens until you do." Buttons: **End semester**, **Remind me later**. Only courses with an expected end date can trigger it.
- **Semester ended:** shown after a semester is ended and nothing new has been added. "Semester ended — CSC 401, CSC 305 are archived. Add your courses for the next semester." Buttons: **Add courses**, × (hide). Disappears when a course is added.

---

## Emails (Ops, v1)

- **Semester start (manual, per school):** Ops sends "new session — here's how to archive your old courses and add new ones" to the lecturers of a school whose session has started. This is the Babcock case.
- **Semester ended (automatic, from the End semester action):** "Your courses are archived — here's how to add your new ones." Sent once per lecturer per ended semester. Skipped if the lecturer has already added or taught a course again.
- **Material request:** section 2.
- No semester-end email is sent from guessed dates.

## Not in v1 (parked)
- Institution calendars keyed by program (full-time/part-time), module or trimester support, and grouping lecturers by school. Revisit once the manual emails show which dates repeat.
- Automatic archiving of any kind.

## Open questions
- Is the department admin the right fallback when no lecturer is linked (material requests)?
- Should "Hide for now" on the setup checklist return every sign-in, or after 7 days?
- Should the end date on add-course be required for schools without an institution account?
- Is 3 days the right lead time for the semester-ending prompt?
