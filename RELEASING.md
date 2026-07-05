Releases go to Maven Central from the GitHub UI: **Actions → Release → Run workflow**. Enter the release version (for example `2.2.0`) and, optionally, the next development version. The [`release`](.github/workflows/release.yml) workflow then:

1. refuses to run if the `v<version>` tag already exists;
2. sets the release version and commits it, and tags `v<version>`;
3. builds, signs and publishes the artifacts to the Sonatype Central Portal under `io.github.vlsi.jqf`;
4. pushes the release commit and tag;
5. creates the GitHub release with generated notes;
6. bumps to the next `-SNAPSHOT` and commits.

The publish step signs every artifact with GPG and uploads source and Javadoc jars, both of which Central requires. It waits until Central has *validated* the bundle, so a broken upload fails the workflow, but it does not block on the final publish, which can take hours and runs asynchronously.

# One-time setup

## Repository secrets

Configure these under **Settings → Secrets and variables → Actions**:

- `CENTRAL_PORTAL_USERNAME`, `CENTRAL_PORTAL_PASSWORD` — a user token from [central.sonatype.com](https://central.sonatype.com) (Account → Generate User Token). The `io.github.vlsi` namespace must already be verified for the account.
- `RELEASE_PGP_PRIVATE_KEY` — the ASCII-armored signing key.
- `RELEASE_PGP_PASSPHRASE` — the passphrase for that key.
- `RELEASE_PGP_SECRET_UPDATE_TOKEN` — a token the PGP Key Maintenance workflow uses to write the refreshed key back into `RELEASE_PGP_PRIVATE_KEY`. Needed only for key maintenance, not for a release.

## Signing key

The signing key lives only in the secrets above; no key material is checked into the repository. The [PGP Key Maintenance](.github/workflows/pgp-key-maintenance.yml) workflow (**Actions → PGP Key Maintenance → Run workflow**) provisions the key, extends the signing subkey before it expires, publishes it to the keyservers, and updates `RELEASE_PGP_PRIVATE_KEY` in place. It wraps the [`vlsi/provision-release-pgp-key`](https://github.com/vlsi/provision-release-pgp-key) reusable workflow; run it whenever the subkey is close to expiry.

# Every release

1. Trigger **Actions → Release → Run workflow** on the branch you release from, and enter the version.
2. Watch the run. If the publish step fails, nothing has been pushed to the repository yet, so fix the cause and run the workflow again.
3. Once the run is green, the artifacts are validated and queued for publishing. They appear on [Maven Central](https://central.sonatype.com/namespace/io.github.vlsi.jqf) within a few hours.

# Local dry run

To reproduce the release build locally without publishing, run the `release` profile with signing skipped:

```bash
mvn -Prelease -Dgpg.skip=true clean verify
```

This produces the source, Javadoc, and binary jars under each module's `target/`, without a signing key and without contacting Central. Drop `-Dgpg.skip=true` if you have a local GPG key and want to check the signatures too.
