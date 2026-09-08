## What

<!-- What does this change do, and why? -->

## Release impact

The release pipeline derives the version bump from every commit that lands on
`main` since the last release. With a squash merge that is this PR's title, so
give it the prefix that matches the intended bump:

| Prefix                                                | Bump  |
| ----------------------------------------------------- | ----- |
| `major:` , any type with `!` (`feat!:`, `fix(api)!:`) | Major |
| `BREAKING CHANGE` in the body                         | Major |
| `feat:`                                               | Minor |
| `fix:`                                                | Patch |
| `chore:`, `docs:`, `refactor:`, `ci:`, `format:`      | none  |

- [ ] PR title uses the prefix that matches the intended bump
- [ ] Breaking changes are called out in the PR body

## Checklist

- [ ] `uv run yamllint .` passes
- [ ] `uv run ansible-lint` passes
- [ ] `uv run molecule test` passes locally for at least one distro
- [ ] New or changed variables are documented as `@var` annotations in
      `defaults/main.yml`
- [ ] `README.md` regenerated (`uv run ansible-doctor`) - never edited by hand
