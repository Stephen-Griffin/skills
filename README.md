# Skills Directory

## Review Changes

Use this skill for substantive reviews of changesets. A changeset may be a GitHub pull request, a local branch, a commit range, staged or unstaged working tree changes, a set of files, or other work that could reasonably be reviewed before merge. The goal is to produce high-signal review feedback, pressure-test it before presenting it, and help the user decide whether and how to post it.

## Resolve Merge Conflicts 

Use this skill to resolve existing Git conflict markers safely. Assume the user is already in the conflicted operation, often an in-progress rebase. Do not initiate, advance, skip, abort, or otherwise control that operation unless the user explicitly instructs you to do that exact action.

## Address PR Feedback

Use this skill to turn reviewer feedback on an existing PR into code changes, folded into the commit each fix actually belongs to rather than tacked on at the tip. Assume the user is already on the branch the feedback applies to. Commit titles are never reworded as a byproduct of a fix, and only change when the user confirms the underlying business requirement changed.
