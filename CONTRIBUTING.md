# Contributing to SafeTrack Pulse

Thanks for your interest in improving SafeTrack Pulse. This guide covers how to
propose changes, our branch and commit conventions, and local setup.

## Code of Conduct

Be respectful and constructive. Harassment or abusive behavior isn't tolerated
in issues, pull requests, or any project space.

## Getting Started

1. **Fork** the repository and clone your fork.
2. Install prerequisites: Node.js `>= 20`, MongoDB `>= 6`, Redis `>= 7`, and a
   Firebase project (Auth + Cloud Messaging).
3. Set up the backend:
   ```bash
   cd server
   cp .env.example .env   # fill in your own credentials
   npm install
   npm run dev
   ```
4. Set up the mobile app:
   ```bash
   cd ../mobile
   npm install
   npx pod-install ios    # iOS only
   npm run ios            # or: npm run android
   ```

## Branch Naming

Use a short, descriptive, kebab-case prefix:

- `feat/<name>` — new feature
- `fix/<name>` — bug fix
- `docs/<name>` — documentation only
- `chore/<name>` — tooling, deps, scaffolding
- `refactor/<name>` — internal change, no behavior difference

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(convoy): add sweep rider role
fix(sos): clamp countdown to 15s on slow devices
docs(readme): document straggler threshold default
```

Keep the subject under ~72 characters and use the body to explain *why*.

## Pull Requests

1. Branch off the latest `main`.
2. Keep PRs focused — one logical change per PR.
3. Make sure the project builds and checks pass:
   ```bash
   npm test
   npm run lint
   ```
4. Update docs (`README.md`, inline comments) when behavior changes.
5. Fill in the PR description: what changed, why, and how to test.
6. Link any related issues (e.g. `Closes #12`).

A maintainer will review, request changes if needed, and squash-merge once
approved.

## Reporting Bugs & Requesting Features

Open an issue with:

- **Bugs:** steps to reproduce, expected vs. actual behavior, device/OS, and
  relevant logs (scrub any PII or location data).
- **Features:** the problem you're trying to solve and your proposed approach.

## Security

Please **do not** open public issues for security vulnerabilities. Report them
privately to the maintainer so they can be addressed before disclosure. Never
include real location data, Medical IDs, or credentials in any report.

## License

By contributing, you agree that your contributions are licensed under the
project's [MIT License](LICENSE).
