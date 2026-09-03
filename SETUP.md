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
| `students/$id` | any signed-in client | the owning teacher (full control); anonymous students may update progress but cannot rename a student or move them to another teacher |

The remaining looseness is deliberate: an anonymous student device can write progress onto any
student record in any class, because there is no per-student credential to check. That is the
tradeoff for password-free student login. If you ever need it tighter, the next step would be a
Cloud Function that mints a custom token per student at code-entry time.

## 3. Data model

```
teachers/{uid}            name, firstName, lastName, email, code, classGrade, createdAt
classCodes/{CODE}         teacherUid, teacherName, grade
students/{pushId}         name, avatar, teacherUid, teacherCode, grade, createdAt
  levels/{A..E}           unlocked, mastery, masteredQuestions{key:tier}, attempts{}
  mult/{t2..t10}          mastery, masteredFacts{f1..f12:tier}, bestCleared, attempts{}
```

The full question set for each level is now generated in the browser instead of being stored on
every student record, which makes student records much smaller.

## 4. Migration

Accounts under the old `users/` node are not read by the new app. Existing teachers need to create
a teacher account again, and add their students by name. The old data is left untouched in the
database if you want to consult it.
