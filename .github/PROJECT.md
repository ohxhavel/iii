# Project management

How work on this repository is tracked. The board holds **execution**; `tech-specs/` remains the
source of truth for roadmap **content**, and `docs/` for user-facing documentation.

## Board

One GitHub Project, board layout, with these fields:

| Field      | Values                                  |
| ---------- | --------------------------------------- |
| `Status`   | Todo · In progress · Blocked · Done      |
| `Priority` | P0 · P1 · P2                            |
| `Area`     | engine · sdk · console · docs · website · ci · crates |

Views:

| View       | Filter               | For                                          |
| ---------- | -------------------- | -------------------------------------------- |
| Roadmap    | `upstream/roadmap`   | Real iii direction, sourced from tech-specs  |
| Fork work  | `fork/work`          | Work specific to this fork                   |
| Design     | `type/design`        | Surfaces awaiting or in design               |
| All        | —                    | Triage                                       |

New issues reach the board through the Project's built-in **Auto-add to project** workflow
(Project → Workflows). That needs no Actions workflow and no token.

## Labels

Two dimensions plus a track. Status lives on the board, never as a label.

| Label                | Meaning                                                |
| -------------------- | ------------------------------------------------------ |
| `upstream/roadmap`   | Real iii direction, sourced from `tech-specs/`          |
| `fork/work`          | Work specific to this fork                             |
| `type/feature`       | New capability or a change to one                      |
| `type/bug`           | Behaviour differs from the docs or the spec            |
| `type/docs`          | Documentation gap, inaccuracy, or drift                |
| `type/spec`          | Epic tracking a `tech-specs/` spec                     |
| `type/design`        | A surface that needs designing                         |
| `type/chore`         | Maintenance, tooling, dependency, or release plumbing   |
| `area/engine`        | `engine/`, `crates/`                                   |
| `area/sdk`           | `sdk/packages/{node,python,rust,go}/`                  |
| `area/console`       | `console/`                                             |
| `area/docs`          | `docs/`                                                |
| `area/website`       | `website/`, `tech-specs/` rendering                    |
| `area/ci`            | `.github/workflows/`, release scripts                  |

Every issue carries exactly one track, one `type/*`, and one `area/*`.

## A spec becomes an epic

1. The spec lands in `tech-specs/<date>-<slug>/` with frontmatter `status: draft`.
2. File a **Tech-spec epic** issue linking that folder, listing its contract docs.
3. File one sub-issue per contract doc and attach them to the epic as sub-issues.
4. When every contract closes, flip the spec's frontmatter to `status: live` and close the epic.

Do not restate spec content in issues — the spec is the document of record, and duplicating it means
two things to keep in sync.

## A design item becomes a design

Design work runs through Claude Design, tracked by the same board:

1. File a **Design item** issue: the surface, the user and job, the states it must cover, and the
   constraints it lives within (existing components, tokens, contracts).
2. Draw it as a Claude Design canvas — multiple artboards on one canvas, one per state or screen.
3. Paste the published canvas URL into the issue's **Design artifact** field and move the issue to
   Done. The canvas is the deliverable; the issue is its record.
4. Implementation issues link back to the design item so a reviewer can find the intended design.

Empty, loading, and error states belong in the original design item, not in a follow-up. The
template lists them as a checklist for that reason.

## Repository prerequisites

Issues are disabled by default on forks. Enable them at **Settings → General → Features → Issues**
before filing anything, and create the labels above (Issues → Labels) so the templates can apply
them — GitHub silently drops a template label that does not yet exist.
