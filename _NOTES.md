# Notes from test suite modernization (2026-08-20)

The suite is one test file, stdlib `test/unit`, real fixtures in `Support/test/data/`.

## Result

`Support/test/test_taskmate.rb`: 7 tests, 7 assertions, 100% passing under Ruby 4.0.6, zero warnings, runnable from any working directory:

```sh
ruby Support/test/test_taskmate.rb
```

Test runs no longer modify the working tree.

## What was broken and since when

| Breakage                                                                                                                                                                                                      | Broken since                         | Fix                                  |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ | ------------------------------------ |
| `require 'lib/taskmate.rb'` needed `.` in the load path                                                                                                                                                       | Ruby 1.9.2 (2010)                    | `require_relative`                   |
| Item text extraction: `(.*)*` always captured `""` under the Oniguruma regular expression engine (the outer repetition matches once more on the empty string, and a capture group reports its last iteration) | Ruby 1.9 (2007)                      | drop the outer star                  |
| Test expectations assumed filesystem-order `Dir.glob` results                                                                                                                                                 | Ruby 3.0 sorts glob results (2020)   | expectations updated to sorted order |
| `str.clone` preserved the frozen state of string literals, so `sub!`/`gsub!` in `Item#initialize` warned under chilled strings and would raise once literals freeze by default                                | Ruby 3.4 warning, Ruby 4.0 direction | `str.dup`                            |
| `@@dir = './test/data'` only resolved when launched from `Support/`                                                                                                                                           | forever                              | `File.expand_path('data', __dir__)`  |
| `test/data/@completed.todo` was a generated artifact accidentally committed by a 2008 test run. Then `test_rebuild_files_exists` deleted it on every run, dirtying the tree                                   | import (2008)                        | deleted deliberately                 |

The text extraction bug is the headline: `Item#text` was empty for every item for roughly fifteen years, which also silently poisoned `Item#==` (it compares texts). Nothing caught it because the suite could not even load. This is the argument for the whole quest series: a syntax-level compatibility sweep can never catch semantic drift. E.g., `(.*)*` parses fine on every Ruby ever made.

## Fossils (identified, deliberately not addressed)

**Four commented-out tests** at the bottom of `test_taskmate.rb` fail with `NoMethodError` when uncommented. They are not aspirational and not environment breakage: they test a pre-rename API. The features all exist under new names. The refactor happened before the bundle's single import commit ("Add Taskmate bundle from Sven Fuchs"), and the old-API tests were commented out instead of ported.

| 2008 test calls                        | Today's equivalent                          |
| -------------------------------------- | ------------------------------------------- |
| `write_files`                          | `rebuild_files`                             |
| `tag_filenames(line)`                  | `filenames_for_item(item)`                  |
| `source_filename(line)`                | `find_item_source(item).filename`           |
| `toggle_completed(:file =>, :line =>)` | `toggle_completed(file, line)` (positional) |

Porting them would be genuinely new coverage. The file-writing paths (`rebuild_files` output content, `toggle_completed` round-trip) are exactly what the seven passing tests do not touch. Feature work, separate decision.

**`Support/taskmate`** (command-line wrapper script) still calls `mate.write_files` which is broken since before the import, roughly eighteen years. It's harmless because it's orphaned. Every `.tmCommand` in the bundle requires `lib/taskmate.rb` directly and uses the current API (`Rebuild Files.tmCommand` calls `mate.rebuild_files`). Dead code carrying a dead call. Deletion candidate for a future dead-code pass.

## Other observations, left unchanged

- `Item#tags` mutates state on read. `@tags.delete(:@completed)` in the getter.
- `Item#attributes=` assigns instance variables via `eval`.
- `rebuild_completed_file` uses `.each` where it appears to mean `.collect`, discarding the block results. This is a latent bug. No test reaches it.
- The marshal cache methods (`marshal_sources`/`unmarshal_sources`) are commented out of `initialize`. It was an abandoned optimization.
