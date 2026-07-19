# FulDC++ website + update server

Static site for [FulDC++](download.html) plus the files the in-client auto-updater fetches.
Served by **GitHub Pages** at `https://fuldcpp.net/`.

## Layout

```
index.html  features.html  download.html  changelog.html   the site
assets/                                                     css + logo/favicon
download/   FulDC-<ver>-x64.zip                             fresh-install package
update/     version.xml  version.xml.sign                   updater manifest + signature
update/updater/ updater_x64_<ver>.zip                       self-update payload
.nojekyll                                                   serve files verbatim
.gitattributes                                              keep version.xml LF-exact
CNAME                                                       custom domain
```

## First-time setup

1. Push this repo, enable **GitHub Pages** (branch = `main`, folder = `/`).
2. Add a `CNAME` file containing your domain and point DNS at GitHub Pages.
3. Turn on **Enforce HTTPS** (the updater uses `https://` URLs).
4. Replace every `fuldcpp.net` placeholder (this README, `update/version.xml`, `download.html`
   is relative so it's fine) with the real domain.

## One-time: create the FulDC++ signing key

The updater only trusts a `version.xml` signed with the private key whose public half is baked
into the client (`airdcpp/airdcpp/core/crypto/pubkey.h`). Do this once, keep `air_rsa` secret.

1. Generate a 2048-bit RSA private key (PEM). The filename **must** be `air_rsa`:
   ```
   openssl genrsa -out air_rsa 2048
   ```
2. Use the client's own tooling to emit the matching `pubkey.h` (byte format must match exactly).
   Run any existing `FulDC.exe` once against the template version.xml:
   ```
   FulDC.exe /sign update\version.xml air_rsa -pubout
   ```
   → writes `pubkey.h` next to `version.xml`.
3. Copy that `pubkey.h` over `airdcpp/airdcpp/core/crypto/pubkey.h` in the client source and
   **rebuild** (`cmake --build --preset=x64-release`). The rebuilt exe is your first real release;
   from now on it trusts only updates signed by `air_rsa`.

> Store `air_rsa` somewhere safe and backed up. Lose it and you can never ship a self-installing
> update to existing clients again (they'd reject anything signed by a new key). Never commit it.

## Publishing a release (see project plan Part C)

1. Build `x64-release` → `compiled/x64-release/windows/FulDC.exe`.
2. Copy `update/version.template.xml` → your build output dir as `version.xml`.

   > **CRITICAL — `<Title>` and `<Message>` MUST be *inside* `<VersionInfo>`** (see the template),
   > not direct children of `<DCUpdate>`. The client's `announceVersion()` searches for them while
   > positioned inside `<VersionInfo>`; if they're outside, the "update available" dialog **never
   > appears** (silently — no error). This is how AirDC++'s own manifest is structured.
3. Run:
   ```
   FulDC.exe /createupdate --resource-directory="...\installer" --output-directory="...\out"
   ```
   This produces `updater_x64_<ver>.zip`, fills in `version.xml` (Build / VersionString / TTH /
   download URL), converts it to **LF** endings and signs it with `air_rsa` → `version.xml.sign`.
4. Commit the three generated files here:
   - `update/version.xml`  (overwrites the template — **must stay LF, byte-exact**)
   - `update/version.xml.sign`
   - `update/updater/updater_x64_<ver>.zip`
   Also drop a fresh-install ZIP in `download/` and bump the version on `download.html`.
5. After pushing, verify: `curl https://fuldcpp.net/update/version.xml | file -` shows no CRLF, and
   an old client offered the higher `Build` accepts and self-installs.

> The private key `air_rsa` is **never** committed. The matching public key is embedded in the
> client (`airdcpp/airdcpp/core/crypto/pubkey.h`); only updates signed by `air_rsa` are trusted.
