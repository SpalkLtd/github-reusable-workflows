Fixtures for the `use-base-ci-tools` self-test. Chosen so a mode bug fails an
assertion in one direction or the other, rather than passing by coincidence:

| path                    | mode   | why |
| ----------------------- | ------ | --- |
| `exec.sh`               | 100755 | exec bit must survive a PR that chmods it 644 |
| `plain.txt`             | 100644 | must NOT gain the exec bit from a PR that chmods it 755 |
| `dir/nested-exec.sh`    | 100755 | per-file modes through the recursive path |
| `dir/nested-plain.txt`  | 100644 | ditto, opposite direction |
| `symlink.txt`           | 120000 | must be refused, not written out as a regular file |
| `dir-with-symlink/nested-link.txt` | 120000 | a symlink NESTED in a restored directory must be refused too |
| `dir-with-symlink/regular.txt`     | 100644 | gives that directory a restorable file, so the refusal is the only reason it fails |

Do not "tidy" the modes — they are the assertions.
