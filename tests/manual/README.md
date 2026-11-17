# Manual test harness

These scripts test the tool against realistic legacy data. They **create courses, users,
questions and attempts**, so only ever run them on a throwaway local site. `create_fixtures.php`
refuses to run unless `wwwroot` contains `localhost`.

They aren't PHPUnit tests. They were used with a local Moodle 4.5 (PHP 8.3, MariaDB 11.4)
laid out like this:

```
testenv/
  moodle/             Moodle 4.5 checkout (MOODLE_405_STABLE), with this plugin in admin/tool/qtypeorder
  moodledata/
  php.sh              runs PHP with the right php.ini
  reset.sh            restores the DB + moodledata snapshot taken after create_fixtures.php
  sync.sh             copies the plugin source into moodle/admin/tool/qtypeorder
  snapshot/           moodle45.sql + moodledata copy
  scripts/            these four files
```

The database is MariaDB on 127.0.0.1:3307, database `moodle45`. `run_tests.sh` also uses a second
database, `moodle45_snap`, holding the snapshot, so that undo can be compared row by row.

## Files

- **create_fixtures.php** creates real shortanswer questions, attempts them through the real quiz and
  question engine, then rewrites them into the exact qtype_order 2012070400 format:
  - 11 questions, 3 quizzes (deferred feedback, interactive, adaptive), and previews
  - autosaves, versions, a random slot, course and system banks, an image, and Unicode text
  - each response is one of: correct, reversed, partly answered with a duplicate position, "I don't know", or unanswered

  It writes `expected.json`.
- **verify.php** runs after a full migration and checks the result in about 450 ways against `expected.json`.
- **post_tests.php** handles the undo round trip compared with the snapshot, a new attempt on converted
  questions, regrade-then-undo refused, and checking a restored course.
- **run_tests.sh** runs the whole suite, about 40 tests, starting from the snapshot. It includes:
  - fault injection: a database trigger that fails one question partway through conversion
  - checksums of every relevant table, to prove simulate and migrate-then-undo change nothing
  - a course backup and restore

```bash
./reset.sh --rebuild     # once: install Moodle, create fixtures, take the snapshot
scripts/run_tests.sh     # any time after that
```
