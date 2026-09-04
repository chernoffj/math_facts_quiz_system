# Edison Math Facts Practice — setup notes

The app is still a single static page (`index.html`) backed by Firebase Auth + Realtime Database.
Two things in the Firebase console **must** be changed before the new version works.

## 1. Enable Anonymous sign-in

Students no longer have accounts. They log in with their teacher's class code and then tap their
name. Under the hood the page signs them in anonymously so the database can still require an auth
token.

Firebase console → **Authentication → Sign-in method → Anonymous → Enable**.

If this is off, students see: *"Student sign-in is not enabled for this project yet…"*.
Email/Password must stay enabled for teacher accounts.

## 2. Publish the new database rules

The data model changed, so the old `users/` rules no longer apply. Copy `firebase-rules.json`
into Firebase console → **Realtime Database → Rules → Publish**.

What the rules enforce:

| Path | Who can read | Who can write |
| --- | --- | --- |
| `classCodes/$code` | any signed-in client, one code at a time (no listing) | only the owning teacher |
| `teachers/$uid` | that teacher only | that teacher only |
| `students` (list) | only as a `orderByChild('teacherUid').equalTo(...)` query | — |
| `students/$id` | any signed-in client | the owning teacher (full control); anonymous students may update progress but cannot rename a student, move them to another teacher, or overwrite a PIN that is already set |

The remaining looseness is deliberate: an anonymous student device can write progress onto any
student record in any class, because there is no per-student credential to check. That is the
tradeoff for password-free student login. If you ever need it tighter, the next step would be a
Cloud Function that mints a custom token per student at code-entry time.

## 2b. Student PINs

After tapping their name, a student has to enter a 4-digit PIN before their account opens.

* **First login** (and any student added before this feature existed): they are asked to choose a
  PIN and type it twice. Nothing is saved unless the two entries match.
* **Every login after that**, including a device that still has a saved session: they type the PIN.
  Five wrong tries locks the screen until they go back and pick a name again.
* **Forgotten PIN:** the teacher clicks **Reset PIN** on that student's card (or on the student
  detail screen). The student then chooses a new PIN at their next login. The roster card shows
  🔒 *PIN set* or 🔓 *No PIN yet* for every student.

The PIN is never stored in the clear. `students/{id}/pinHash` holds a SHA-256 digest of
`edison-math-pin-v1|{studentId}|{pin}`, so the same 4 digits produce a different value for each
student, and the digest cannot be typed back in as a PIN. Four digits is a small space, so this
protects against classmates poking at each other's accounts, not against a determined attacker
with database access.

## 3. Data model

```
teachers/{uid}            name, firstName, lastName, email, code, classGrade, createdAt
classCodes/{CODE}         teacherUid, teacherName, grade
students/{pushId}         name, avatar, teacherUid, teacherCode, grade, createdAt, pinHash
  levels/{A..E}           unlocked, mastery, masteredQuestions{key:tier}, attempts{}
  mult/{t2..t10}          mastery, masteredFacts{f1..f12:tier}, bestCleared, bestFloors,
                          bestFloorsOutOf, attempts{}
```

`bestCleared` is the most facts cleared in a single climb; `bestFloors` is the highest the climber
reached in a single climb (it can be a half number), and `bestFloorsOutOf` says how tall the tower
was when that best was set. Records written before `bestFloors` existed still load — it defaults to
0 and fills in on the next climb. Records written when the tower was 12 floors tall have no
`bestFloorsOutOf`; those default to 12, so an old best still reads as `12/12` rather than being
silently rescored against the taller tower. A new climb takes over the record when it is at least as
good *as a fraction of its own tower*.

## 3b. Sky Climber rules

Every climb is **exactly 20 questions on a 20-floor tower** — one floor per question, so a flawless
climb reaches the roof.

**The question set.** The first 12 questions are the whole table (`×1` to `×12`) in random order.
The last 8 are repeats, chosen from that opening pass:

* every fact the student got **wrong** comes back first, slowest miss first;
* the remaining slots are filled with the **slowest** correct answers, slowest first;
* so if nothing was missed, the 8 repeats are simply the 8 slowest facts.

The same fact is never asked twice in a row.

**The climb.** Each question is scored on its own:

| Answer | Climb | Fact |
| --- | --- | --- |
| Correct in under 3s (mastery speed) | full floor | counts as cleared for this run |
| Correct in under 6s | half a floor | not cleared |
| Correct but slower, or wrong | no climb | not cleared |

A repeat of an already-cleared fact still earns its floor, because the floor belongs to the question
rather than to the fact. `Facts Cleared` is still out of 12 and is what module mastery is based on;
`Floors Climbed` is out of 20 and is the score for the run.

The full question set for each level is now generated in the browser instead of being stored on
every student record, which makes student records much smaller.

## 3c. How an answer gets graded

This applies to both run modes. A correct answer is accepted the moment it is typed. A wrong one is
only judged on sight once it is as long as the right answer, because anything shorter might still be
half typed — so a student who types `5` for `7 × 3` has not answered yet. Three things make sure that
can never strand a run:

* the green **✓ Answer** button (and the Enter key) grades whatever is in the box, at any time;
* an answer left untouched in the box for 12 seconds is graded on its own — well past every speed
  tier, so it costs no credit that was not already lost;
* the number pad refuses a second leading zero, so the box can never fill with zeros that are
  shorter than the answer they are checked against.

Results are rendered **before** the round is saved, and the save is raced against a 12-second
timeout. A Firebase write stays pending forever on an offline device rather than failing, so waiting
on it used to hide the results screen entirely; now the student always sees their score, and a save
that does not land offers a **Try again** button on the results banner.

## 4. Migration

Accounts under the old `users/` node are not read by the new app. Existing teachers need to create
a teacher account again, and add their students by name. The old data is left untouched in the
database if you want to consult it.

Student records created before PINs existed simply have no `pinHash`; those students are asked to
choose one the next time they log in. Nothing needs to be migrated by hand.
