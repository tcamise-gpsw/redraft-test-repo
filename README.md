# redraft-test-repo

Test fixture workspace for [ReDraft](https://github.com/tcamise-gpsw/redraft).

This repository is consumed as a **git submodule** at `test-fixtures/` inside the
main ReDraft repo. Playwright's local-mode E2E suite seeds its throwaway workspace
by rsyncing from this directory.

---

## Structure

```
.
├── api-design-v2.md                              # root-level doc (under review)
├── getting-started.md                            # simple intro doc
├── docs/
│   ├── auth-overhaul.md                          # used by editing + comment E2E tests
│   ├── architecture.md                           # system overview with Mermaid diagrams
│   └── deployment.md                             # build, publish, CI guide
├── rfcs/
│   ├── rfc-001-pagination.md                     # accepted RFC
│   └── rfc-002-error-responses.md                # draft RFC
└── .redraft/
    └── comments/
        └── api-design-v2.comments.json           # sidecar → marks api-design-v2.md "Under Review"
```

Documents that have a matching `.redraft/comments/*.comments.json` sidecar appear in
the **Under Review** section of the ReDraft sidebar. All others appear under
**Documents**.

---

## E2E dependencies

The local-mode Playwright suite (`e2e/local-mode.spec.ts`) relies on these specific
paths being present:

| Path | Required by |
|------|-------------|
| `api-design-v2.md` | tree renders "Under Review" entry |
| `.redraft/comments/api-design-v2.comments.json` | above entry shows in Under Review |
| `docs/auth-overhaul.md` | editing + comment creation tests |

`docs/auth-overhaul.md` must contain the phrase **"current session-cookie
authentication"** — the comment E2E selects that text to open the comment form.

---

## Adding fixtures

1. Add a `.md` file anywhere in the tree.
2. To mark it "Under Review", add a corresponding sidecar at
   `.redraft/comments/<mirrored-path>.comments.json`:
   ```json
   { "version": 1, "comments": [] }
   ```
3. Commit and push to `main`.
4. In the ReDraft repo, `cd test-fixtures && git pull && cd .. && git add test-fixtures`
   and commit the updated submodule pointer.

---

## Updating the submodule pointer in ReDraft

```bash
# from the redraft repo root
cd test-fixtures
git pull
cd ..
git add test-fixtures
git commit -m "chore(fixtures): update test-fixtures submodule"
git push
```
