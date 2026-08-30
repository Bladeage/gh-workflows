# Self-hosted CI auf den QNAP-Runnern — Checkliste

Private Repos laufen auf per-Repo self-hosted QNAP-Runnern (`runs-on: [self-hosted, qnap]`,
ephemer, global via flock = **nur 1 Job gleichzeitig**, persistenter toolcache). Grund:
das GitHub-Free-Kontingent ist erschöpft → **hosted-Jobs starten nicht** (auch der
Actions-Cache-Service ist gesperrt). Öffentliche Repos haben unbegrenzte Minuten → bleiben
hosted. **Jeden neuen self-hosted-Workflow gegen diese Liste bauen:**

## Die 6 Runner-Grenzen + Fixes

1. **Docker-Build**: buildx `docker-container`-Treiber scheitert am Kernel 5.10
   (`fchmodat2`). → `docker/setup-buildx-action` mit **`driver: docker`** (Daemon-BuildKit,
   Host-runc), **kein** `cache-from/to: type=gha`. Runner ist x86_64 → Images sind amd64.
2. **Android**: kein SDK im Image. → `actions/setup-java@v4` (17/21) + `android-actions/setup-android@v3`.
   Persistentes `/opt/android-sdk`-Volume je Repo (`ANDROID_SDK_ROOT` ist gesetzt).
   **Kein KVM** → nur `lint` + `testDebugUnitTest`, **niemals** connected/instrumented Tests.
3. **Service-Container** (`services:`): gehen nicht (Runner im Container → „network null").
   → ephemere Sibling-Container im Netz **`ci-services`**:
   `docker run -d --rm --network ci-services --name pg-${{ github.run_id }} …`, Hostname =
   Container-Name, `if: always()`-Teardown. Runner muss ans Netz gehängt sein (qnap).
4. **Alt-Interpreter**: Image-Basis Python **3.8** / node **v10** → moderne Syntax bricht.
   → `actions/setup-python@v5` / `actions/setup-node@v4` **vor** den Skripten.
5. **Windows/macOS**: kein self-hosted-Äquivalent. → Job hinter Repo-Variable
   **`HOSTED_OK`**-Gate (`if: ${{ vars.HOSTED_OK == 'true' }}`, schläft bis Billing zurück).
   **Flutter**: kein subosito-`cache: true` → clone-if-missing ins `/opt/flutter`-Volume
   (`FLUTTER_ROOT`), `PUB_CACHE=/opt/flutter-pub`.
6. **cgroup v1 → nproc nicht virtualisiert**: Tools sehen 12 Host-Threads statt der
   Container-CPUs → jest/vitest/pytest-xdist/gradle spawnen zu viele Worker → OOM/Thrash
   (**hält den flock → blockiert ALLE Repos**). → Parallelität **immer hart deckeln**:
   `jest --maxWorkers=2` (bzw. `maxWorkers` in jest.config.js; `1` gegen einen Emulator),
   `vitest maxWorkers/maxThreads=2`, `pytest -n2`, Gradle `org.gradle.workers.max=2`
   (+ `-Xmx4g` + `daemon=false` via `$HOME/.gradle/gradle.properties`, **WRITE nicht append**).
   AAB-Build-Repos brauchen **10–12 GB** mem_limit (bei qnap anmelden, nicht erst beim roten Build).

## Zwei Zeitbomben

- **Nie `cache:`-Inputs an setup-Actions** (`setup-node cache: npm`, `setup-python cache: pip`,
  `setup-gradle`) — die reden mit dem gesperrten Actions-Cache-Service → rot/hängt.
  Persistenz **nur** über qnap-Volumes.
- **Disk-Wachstum**: driver:docker-Layer + Gradle/Android/Flutter-Volumes wachsen unbegrenzt.
  Prune-Policy mit qnap vereinbaren, bevor die volle Platte den flock-Job killt.

## Sonstiges
- 2-Job-Workflows serialisieren hart (1 Runner/Repo + flock) → Job 2 wartet lang in
  „Set up runner"; bei Re-Push kann er canceln (kein echter Fehler). Wenige Jobs/Workflow bevorzugen.
- Neues Repo braucht self-hosted-CI → qnap legt einen per-Repo-Runner an (gleiches Muster).
- GHCR-Push: `docker/login-action` + `GITHUB_TOKEN`, `permissions: packages: write`.
  Erster echter Push ist outward → mit OK.
