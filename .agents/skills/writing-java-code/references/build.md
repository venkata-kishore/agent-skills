# Build

- Gradle with the Kotlin DSL. An included `build-logic` build holds the convention plugins; module builds apply them and stay short.
- Every version lives in the version catalog (`gradle/libs.versions.toml`), never inline in a build file.
- Latest stable only, pinned exactly. Check the registry's own metadata, not a search page.
- Gate tasks fail the build on a pre-release version (with a small explicit allowlist) and on an unpinned version.
- Dependency locking plus dependency verification metadata with sha256 checksums.
- After any dependency change, regenerate both in an empty Gradle user home: run the project's lock task (one that resolves every configuration) with `-g "$(mktemp -d)" --write-locks --write-verification-metadata sha256`. A warm cache satisfies parent POMs without recording them, so regenerating in your normal home leaves entries out and the next clean machine fails.
- A verification failure is a finding: find out why the artifact changed. Never refresh the metadata just to make it pass.
- Any new dependency needs the user's approval and an open-source licence.
