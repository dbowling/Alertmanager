# Alertmanager for macOS — v2 plan

Status: proposed (2026-10-04). Coordinator: Fable. Implementer: Sonnet.
Scope of this document: what to build next, in what order, and how each
step is verified. It is written so that an implementer with no prior
context can pick up any single milestone and finish it.

## 1. Goals

1. Modern toolchain: Swift 6 language mode, Xcode 26.x on CI, deployment
   target macOS 26, verified to run on macOS 27 (the current major).
2. Essential automated tests that run on every pull request in GitHub
   Actions, fast enough to iterate on (target: under 10 minutes wall
   clock for the required job).
3. Multiple Alertmanager endpoints, including ones that live inside a
   Kubernetes cluster and are not exposed outside it. For those, the app
   runs and supervises a background `kubectl proxy` process bound to a
   specific kubeconfig context, and addresses the Alertmanager service by
   an explicit namespace that may differ from the context's default.
4. Keep the existing JSON configuration working (backwards compatible),
   adding new keys rather than renaming old ones.

Non-goals for v2: a graphical settings editor, App Store distribution,
Intel-only concerns beyond what the macOS 26 target gives for free.

## 2. Where the app is today (facts from the tree at `f608c45`)

- ~500 lines of Swift 5 / AppKit, created 2019. `@NSApplicationMain`,
  a storyboard, `NSStatusItem` + `NSPopover` wrapping a `WKWebView` that
  renders HTML produced by Stencil templates (`ViewController.swift`).
- `Alerts.swift` is a singleton that reads `~/.alertmanager.json`,
  polls each Alertmanager's `/api/v2/alerts/groups` on a `Timer`, and
  appends results from concurrent `URLSession` callbacks into shared
  arrays with no synchronisation (a real data race; `loadErrors` and
  `alertGroups` are mutated from several completion handlers).
- Notifications use `NSUserNotification`, deprecated since macOS 11.
- Build settings: `MACOSX_DEPLOYMENT_TARGET = 10.15`, `SWIFT_VERSION =
  5.0`, `objectVersion = 52`, ad-hoc signing (`CODE_SIGN_IDENTITY = "-"`),
  empty entitlements (no App Sandbox), `NSAllowsArbitraryLoads = true`.
- One dependency: Stencil 0.13.1 (via SwiftPM in the Xcode project).
- CI: `.github/workflows/build.yaml` archives on `macos-latest` with
  `actions/checkout@v1`, no tests. Release upload goes through a hand
  rolled `upload.sh`.
- The GitHub repo `dbowling/Alertmanager` is a public fork of
  `ricoberger/Alertmanager`, default branch `master`, zero workflow runs
  so far, no pull requests.

## 3. Preliminary digging: blockers and decisions

No hard blocker was found. Each requirement was checked; the items below
need a one-time action or a decision, and the plan proceeds with the
stated default unless the owner says otherwise.

| # | Requirement | Finding | Default decision |
|---|-------------|---------|------------------|
| 1 | Latest macOS | macOS 27 "Golden Gate" shipped 2026-09-14 and is Apple-silicon only. GitHub's `macos-latest` label now resolves to macOS 26 with Xcode 26.6 (Swift 6.3.3). An Xcode 27 image exists only as a public preview under the `xcode-27` label. | Deployment target macOS 26. Required CI job pins `runs-on: macos-26` with Xcode 26.6. An advisory, non-blocking job runs the same tests on `xcode-27` so we see macOS 27 runtime results without depending on a preview image. Build universal (arm64 + x86_64) since macOS 26 still supports Intel. |
| 2 | CI on PRs | Actions is disabled by default on forks until someone clicks "I understand my workflows, go ahead and enable them" in the repo's Actions tab. This cannot be done from a session. Public repo, so macOS runner minutes are free. | Owner enables Actions once. Milestone 1 is the first PR that exercises it; the coordinator checks that a run appears. |
| 3 | Up-to-date libraries | Stencil's last release is 0.15.1 (July 2022). It still builds with Swift 6.x compilers and the Swift Package Index lists it compatible through Swift 6.3, but it is effectively unmaintained and not `Sendable`-annotated. | Keep Stencil 0.15.1 for config compatibility, isolate it behind a `TemplateRenderer` protocol in the core package, and run it on a single actor so strict concurrency is satisfied. Re-evaluate replacement in v3. |
| 4 | Language version | Xcode 26.6 bundles Swift 6.3.3; Swift 6.4.0 is current on swift.org (Linux) and in Xcode 27. | `swift-tools-version: 6.2`, `swiftLanguageModes: [.v6]`. Everything in the core package must be strict-concurrency clean. |
| 5 | Implementer tooling | Implementer sessions run on Linux (Ubuntu 24.04, x86_64) with no Xcode. `download.swift.org` is reachable, so a Linux Swift toolchain can be installed in-session and `swift test` can run for Foundation-only code. `xcodebuild` is CI-only. | Put all logic in a SwiftPM package (`AlertmanagerCore`) that builds on Linux and macOS. Keep the Xcode app target thin (AppKit/SwiftUI/WebKit/UserNotifications only) so CI is the only place it needs to compile. |
| 6 | Xcode project maintenance | Editing `project.pbxproj` by hand is error-prone and cannot be verified outside CI. XcodeGen's last release is 2.46.0 (July 2024). | Rewrite the project once to `objectVersion = 77` with file-system-synchronized groups (Xcode 16+ format). After that, adding Swift files under `App/` needs no pbxproj edit. No generator dependency. |
| 7 | kubectl proxy | `kubectl proxy` honours the global `--context` and `--kubeconfig` flags, listens on `127.0.0.1:8001` by default, `--port=0` picks a free port, and prints `Starting to serve on 127.0.0.1:<port>` to stdout once bound. Services are reached at `/api/v1/namespaces/<ns>/services/[https:]<svc>[:<port>]/proxy/...`, so the namespace is part of the URL and independent of the context's default namespace. | Design in section 6. Not a blocker. |
| 8 | GUI PATH | A GUI app launched from Finder gets `PATH=/usr/bin:/bin:/usr/sbin:/sbin`. `kubectl` and its exec credential plugins (`gke-gcloud-auth-plugin`, `aws`, `kubelogin`, `az`) are usually in `/opt/homebrew/bin` or `/usr/local/bin`. | Resolve the login shell's PATH once at startup and pass it to child processes; allow an explicit `kubectlPath` and per-cluster `env` overrides. |
| 9 | Sandbox / signing | Spawning `kubectl` and reading `~/.kube/config` is incompatible with App Sandbox. The project already runs unsandboxed with ad-hoc signing. Notarization needs a paid Apple Developer account. On macOS 26/27 an unsigned download must be allowed via System Settings > Privacy & Security ("Open Anyway"). | Stay unsandboxed and ad-hoc signed. Enable Hardened Runtime (harmless here). Document the Gatekeeper step in the README. Notarization is an owner decision outside this plan. |
| 10 | Alertmanager API | Alertmanager 0.34.1 (2026-09-17) is current. API v2 `/alerts/groups` is unchanged; 0.33 added `receiver_matchers` filters. | Keep API v2. Capture a fixture from 0.34 for decoding tests. |

## 4. Target architecture

```
Alertmanager/                      repo root
├── Package.swift                  SwiftPM manifest (tools 6.2, language mode 6)
├── Sources/AlertmanagerCore/      Foundation-only, builds on Linux + macOS
│   ├── Config/                    Config model, decoding, validation, defaults
│   ├── API/                       AlertmanagerClient, models, URL building
│   ├── Kubernetes/                KubectlProxy, KubectlProxyManager, ProcessRunner
│   ├── Rendering/                 TemplateRenderer (Stencil impl), Severity
│   └── Store/                     AlertStore actor (poll loop, merge, "new alert" detection)
├── Tests/AlertmanagerCoreTests/   Swift Testing (`import Testing`)
│   └── Fixtures/                  JSON fixtures, fake-kubectl.sh
├── App/                           Xcode app target, thin
│   ├── AlertmanagerApp.swift      @main, NSApplicationDelegateAdaptor
│   ├── StatusItemController.swift NSStatusItem (left click: popover, right click: menu)
│   ├── AlertsPopoverView.swift    SwiftUI view hosting WebView(HTML)
│   ├── WebView.swift              NSViewRepresentable around WKWebView
│   ├── Notifier.swift             UserNotifications wrapper
│   ├── Assets.xcassets, Info.plist, Alertmanager.entitlements
├── Alertmanager.xcodeproj         objectVersion 77, synchronized groups, depends on ../ (local package)
├── .github/workflows/ci.yml       PR + push: test core, build app, lint
├── .github/workflows/release.yml  tag v*: archive, zip, GitHub release
├── .github/dependabot.yml         github-actions + swift
├── Makefile                       test, build, project, release-*
└── README.md
```

Design rules the implementer must follow:

- `AlertmanagerCore` imports only `Foundation` (and Stencil). No AppKit,
  WebKit, UserNotifications, SwiftUI. This is what keeps it testable on
  Linux and keeps the required CI job fast.
- Concurrency: `actor` types for mutable shared state (`AlertStore`,
  `KubectlProxyManager`, `StencilRenderer`). `async/await` with
  `URLSession.data(for:)`; no completion-handler APIs. All model structs
  are `Sendable`.
- Dependency injection via small protocols so tests do not touch the
  network or spawn real kubectl: `HTTPTransport` (default wraps
  `URLSession`), `ProcessLaunching` (default wraps `Foundation.Process`),
  `ReadinessProbe`, `Clock` (use `ContinuousClock` from the standard
  library; tests pass a test clock).
- The app target is UI glue only: it observes `AlertStore` and
  `KubectlProxyManager` state and renders it.

## 5. Configuration schema (v2, backwards compatible)

File stays at `~/.alertmanager.json`. Existing keys keep their meaning
and defaults. `silenced` / `inhibited` accept both JSON booleans and the
legacy `"true"`/`"false"` strings. New keys:

```json
{
  "refreshInterval": 60,
  "severityLabel": "severity",
  "titleTemplate": "[{{ name | uppercase }}] {{ labels['alertname'] }}",
  "alertTemplate": "{{ annotations['message'] }}",

  "kubernetes": {
    "kubectlPath": "/opt/homebrew/bin/kubectl",
    "clusters": [
      {
        "name": "prod-eu",
        "context": "gke_myproj_europe-west1_prod",
        "kubeconfig": "~/.kube/config",
        "port": 0,
        "env": { "CLOUDSDK_CORE_PROJECT": "myproj" }
      }
    ]
  },

  "alertmanagers": [
    {
      "name": "PUBLIC",
      "url": "https://alertmanager.example.com",
      "authType": "basic",
      "authUsername": "user",
      "authPassword": "pass"
    },
    {
      "name": "PROD-EU",
      "silenced": false,
      "inhibited": false,
      "kubernetes": {
        "cluster": "prod-eu",
        "namespace": "monitoring",
        "service": "alertmanager-main",
        "port": "web",
        "scheme": "http"
      }
    }
  ]
}
```

Rules:

- An Alertmanager entry has exactly one of `url` or `kubernetes`.
  Validation reports a clear error naming the entry otherwise.
- `kubernetes.cluster` must reference a `kubernetes.clusters[].name`.
- `kubernetes.port` is a service port name or number; optional. `scheme`
  is `http` (default) or `https`.
- `clusters[].port` is the local proxy port; `0` (default) means pick a
  free port. `kubeconfig` is optional and may contain `~`.
- `clusters[].context` is required. The namespace is never taken from
  the context; it always comes from the Alertmanager entry.
- Effective base URL for a Kubernetes entry:
  `http://127.0.0.1:<proxyPort>/api/v1/namespaces/<namespace>/services/<scheme-prefix><service>[:<port>]/proxy`
  where `scheme-prefix` is empty for `http` and `https:` for `https`.
  Path segments are percent-encoded. This base URL is what the
  `titleTemplate` `url` variable receives.
- Auth headers are not sent for Kubernetes entries; kubectl supplies
  cluster credentials. If an entry sets both, validation fails.

## 6. kubectl proxy background service

One `KubectlProxy` per configured cluster, owned by the
`KubectlProxyManager` actor. Lifetime is tied to the app process.

State machine per proxy:

```
stopped -> starting -> ready(port) -> (exited/crashed) -> backoff(n) -> starting ...
                    \-> failed(reason)   // non-retryable: kubectl not found, context unknown
```

Startup sequence:

1. Resolve the `kubectl` binary: `kubernetes.kubectlPath` if set; else
   the first of `/opt/homebrew/bin/kubectl`, `/usr/local/bin/kubectl`,
   `~/.rd/bin/kubectl`, `/Applications/Docker.app/Contents/Resources/bin/kubectl`
   that exists; else `command -v kubectl` via the user's login shell
   (`$SHELL -l -c`, 5 s timeout). Failure is `failed(kubectlNotFound)`.
2. Build the environment: inherit the process env, override `PATH` with
   the login shell's PATH (captured once at startup, cached), set
   `KUBECONFIG` if `kubeconfig` is given, then apply `clusters[].env`.
3. Spawn `kubectl proxy --context=<ctx> --address=127.0.0.1 --port=<port>
   --accept-hosts='^localhost$,^127\.0\.0\.1$,^\[::1\]$'`. Do not pass
   `--namespace`; it is irrelevant to the proxy and would mislead readers.
4. Read stdout line by line. The first line matching
   `Starting to serve on 127.0.0.1:(\d+)` yields the port. Then probe
   `GET http://127.0.0.1:<port>/version` until HTTP 200 (10 s budget).
   Transition to `ready(port)`. stderr is retained (last 20 lines) for
   the status menu and error reporting.
5. On process exit while in `ready`: restart with exponential backoff
   1, 2, 4, ..., 60 s (capped), unless the manager is shutting down.
   Exit before readiness with a stderr containing `context was not
   found` or `error: unknown command` is `failed` (no retry); other
   pre-ready exits retry up to 5 times then `failed`.
6. Shutdown: `SIGTERM`, wait up to 3 s, then `SIGKILL`. Hooked into
   `applicationWillTerminate` and the manager's `deinit`.
7. Orphan protection: write `{pid, startTime, clusterName}` records to
   `~/Library/Application Support/Alertmanager/proxies.json` on spawn.
   On launch, for each record whose pid is alive and whose executable
   basename is `kubectl`, send `SIGTERM`, then clear the file.

Alertmanager clients for Kubernetes entries await `ready(port)` from
the manager before each poll; while the proxy is not ready, the entry
reports a load error of the form `PROD-EU: kubectl proxy starting` or
`PROD-EU: kubectl proxy failed: <reason>` in the popover, not a crash.

Security notes: the proxy binds to loopback only and keeps kubectl's
default `--reject-paths` (pod exec/attach). The app never reads cluster
credentials itself. Config is user-owned; no secrets are added by this
feature beyond what the user already stores in the file.

## 7. Testing strategy

Framework: Swift Testing (`import Testing`), run with `swift test` in
the package. XCTest is not used for new tests. Tests must not hit the
network, the real filesystem outside a temp dir, or a real kubectl.

Essential tests (these are the acceptance gate for milestones 1 and 3):

Config
- Decodes the legacy example from the README unchanged; defaults match
  the current `Models.swift` values exactly.
- `silenced: "true"` (string) and `silenced: true` (bool) both decode.
- Kubernetes entry decodes; `url` + `kubernetes` together fails with an
  error naming the entry; unknown `cluster` fails; missing `context`
  fails; `~` in `kubeconfig` expands.

API
- Decodes `Fixtures/alerts_groups_v0.34.json` (captured from a real
  Alertmanager 0.34.1) into `[AlertGroup]`; group `startsAt` is the
  newest alert's; groups sort newest first across two Alertmanagers.
- URL building: plain entry appends `/api/v2/alerts/groups?silenced=…&inhibited=…`;
  Kubernetes entry produces the exact proxy URL for `http` and `https`
  schemes, named and numeric ports, and a namespace needing encoding.
- Basic and bearer auth headers are set for plain entries and absent
  for Kubernetes entries (via a recording `HTTPTransport`).
- A transport error for one Alertmanager does not drop results from
  another (regression test for the current race).

Kubernetes proxy
- Argument construction: context, port, address, accept-hosts; `KUBECONFIG`
  env set only when configured; `PATH` overridden; `env` merged.
- Readiness: a fake `ProcessLaunching` emits `Starting to serve on
  127.0.0.1:43123` and a fake `ReadinessProbe` succeeds; state becomes
  `ready(43123)`.
- Crash after ready restarts with backoff 1, 2, 4 s using a test clock
  and stops when `shutdown()` is called.
- `context was not found` on stderr before ready yields `failed`, no retry.
- `shutdown()` sends terminate, then kill after the grace period
  (fake process records the signals).
- Integration (macOS CI only, tagged `.tags(.integration)`): spawn
  `Fixtures/fake-kubectl.sh` for real through `Foundation.Process`; the
  script prints the ready line and sleeps; assert the port is parsed and
  the process is terminated on shutdown.

Rendering
- Default `titleTemplate` and `alertTemplate` render the README example
  context to the exact expected strings.
- Severity mapping for info/warning/error/critical/unknown/no-label.

App target: no unit tests in v2 beyond compiling; the app is verified
by the CI build and a manual smoke checklist (section 10).

## 8. CI design

`ci.yml` (on `pull_request` and `push` to `master`):

| Job | Runner | Steps | Blocking |
|-----|--------|-------|----------|
| `core-tests` | `macos-26` | `sudo xcode-select -s /Applications/Xcode_26.6.app`; `swift --version`; `swift build`; `swift test --parallel` | yes |
| `lint` | `macos-26` | `swift format lint --strict --recursive Sources Tests App` (built into the toolchain, no install) | yes |
| `app-build` | `macos-26` | `xcodebuild -project Alertmanager.xcodeproj -scheme Alertmanager -configuration Release -destination 'generic/platform=macOS' CODE_SIGN_IDENTITY=- build`; upload `.app` zip as artifact | yes |
| `core-tests-linux` | `ubuntu-24.04` | `swift:6.2` container; `swift test` | yes (cheap, catches platform-specific code sneaking into core) |
| `macos-27-preview` | `xcode-27` | same as `core-tests` + `app-build` | no (`continue-on-error: true`) |

Caching: `actions/cache` keyed on `Package.resolved` for `.build`.
Actions pinned by major (`actions/checkout@v5`), Dependabot keeps them
current. `concurrency` cancels superseded runs on the same PR.

`release.yml` (on tag `v*`): archive with `xcodebuild archive`, zip the
`.app`, create the GitHub release with `softprops/action-gh-release`
and attach the zip plus a SHA-256 file. `upload.sh` is deleted.

Branch protection on `master` (owner action): require `core-tests`,
`lint`, `app-build`, `core-tests-linux`.

## 9. Toolchain and dependency targets

| Item | Target |
|------|--------|
| macOS deployment target | 26.0 |
| Verified runtime | macOS 26 (required CI), macOS 27 (advisory CI + owner's machine) |
| Xcode | 26.6 pinned in CI; project `LastUpgradeCheck = 2660` |
| Swift | tools 6.2, language mode 6, strict concurrency complete |
| Stencil | 0.15.1 (`from: "0.15.1"`) |
| Alertmanager API | v2 (fixture from 0.34.1) |
| kubectl | any recent client; tested against the `Starting to serve on` contract, stable since kubectl 1.x |
| GitHub Actions | `actions/checkout@v5`, `actions/cache@v4`, `actions/upload-artifact@v4`, `softprops/action-gh-release@v2` |

## 10. Milestones (one pull request each, in order)

Each milestone lists its acceptance criteria. A milestone is done when
CI is green on the PR, the criteria are met, and the coordinator has
reviewed the diff.

### M1 — Core package, tests, and CI (foundation)

- Add `Package.swift` with `AlertmanagerCore` and its test target.
- Move `Config`, `Alertmanager`, `AlertGroup`, `Alert`, `Receiver`,
  `AlertStatus` into `Sources/AlertmanagerCore`, make them `Sendable`,
  keep decoding behaviour identical (tests prove it).
- Add `AlertmanagerClient` (async, `HTTPTransport` injected) and the URL
  builder for plain entries.
- Add `StencilRenderer` actor behind `TemplateRenderer`.
- Add the Config, API (plain entries), and Rendering tests from section 7
  with fixtures.
- Add `ci.yml` with `core-tests`, `lint`, `core-tests-linux`. Add
  `dependabot.yml`. Remove `build.yaml`.
- Acceptance: `swift test` passes on macOS and Linux CI; `swift format
  lint` clean; a workflow run is visible on the PR (confirms the owner
  enabled Actions).

### M2 — Modern app shell on the core package

- Rewrite `project.pbxproj` to objectVersion 77 with synchronized
  groups for `App/`, deployment target 26.0, Swift 6 mode, Hardened
  Runtime on, local package dependency on `AlertmanagerCore`, universal
  architectures. Delete the storyboard.
- `@main` SwiftUI `App` with `NSApplicationDelegateAdaptor`;
  `StatusItemController` keeps the current click semantics (left:
  popover, right: menu with Open Configuration, Reload Configuration,
  Quit).
- Popover content is a SwiftUI view wrapping `WKWebView`
  (`NSViewRepresentable`); link clicks open in the default browser.
- `AlertStore` actor replaces the `Alerts` singleton: structured poll
  loop with `Task.sleep`, per-Alertmanager results merged atomically,
  "new alerts" detection preserved.
- Notifications via `UserNotifications` with permission request on
  first launch.
- Keep `NSAllowsArbitraryLoads = true` for v2: many Alertmanagers are
  reached over plain `http://` on internal networks, and ATS exceptions
  cannot be added per config entry at runtime. Tightening this is a v3
  item, noted in the README's security section.
- Add `app-build` job to `ci.yml`.
- Acceptance: app builds in CI; manual smoke checklist passes on the
  owner's macOS 27 machine: launch, icon appears, left click shows
  alerts from a public Alertmanager, right click menu works,
  notification arrives for a new alert, quit is clean.

### M3 — Kubernetes clusters and kubectl proxy supervisor

- Config schema additions from section 5 with validation and tests.
- `ProcessLaunching`, `KubectlProxy`, `KubectlProxyManager` per section
  6, with all Kubernetes tests from section 7 including the integration
  test with `fake-kubectl.sh`.
- Kubernetes URL building in `AlertmanagerClient`; clients wait for
  readiness; proxy state surfaces as load errors in the popover.
- Right-click menu gains a "Kubernetes" submenu: one row per cluster
  showing `Ready :port` / `Starting` / `Failed: reason`, plus "Restart"
  per cluster and "Show kubectl log" (opens a temp file with the
  retained stderr).
- README documents the feature, the PATH behaviour, and an end-to-end
  example with `kubectl get svc -n monitoring`.
- Acceptance: all tests green; manual check against a real private
  cluster: alerts load through the proxy, killing the kubectl process
  externally triggers an automatic restart, quitting the app leaves no
  `kubectl proxy` behind (`pgrep -fl "kubectl proxy"` is empty).

### M4 — Release pipeline and docs

- `release.yml` per section 8; delete `upload.sh`; update `Makefile`
  (`make test`, `make build`, `make release-{major,minor,patch}`).
- README: installation on macOS 26/27 including the Gatekeeper "Open
  Anyway" step, full configuration reference, troubleshooting section.
- Acceptance: pushing a `v2.0.0-rc.1` tag produces a GitHub release with
  a zip that launches on the owner's machine.

### M5 (optional) — Linux end-to-end proxy test

- A `kind` cluster on `ubuntu-24.04` with a minimal Alertmanager
  deployment; run the real `KubectlProxyManager` against it from the
  core package's integration tests. Non-blocking job. Only worth doing
  if M3 shows contract drift risk.

## 11. Coordinator / implementer protocol

- One milestone per PR, branched from `master`, named `v2/m<N>-<slug>`.
  No milestone starts before the previous one is merged, except M4
  which may run in parallel with M3.
- The implementer installs a Linux Swift toolchain in-session (swiftly
  or the Ubuntu 24.04 tarball from swift.org) and runs `swift build`
  and `swift test` for `AlertmanagerCore` before every push. `App/` and
  the pbxproj are verified only by CI, so pbxproj changes are confined
  to M2 and kept minimal.
- Each PR description states: what changed, which section 7 tests were
  added, what was verified locally, what only CI verified, and any
  deviation from this plan with the reason.
- The coordinator reviews every PR for: core package purity (no
  AppKit imports), concurrency correctness (no `nonisolated(unsafe)`,
  no `@unchecked Sendable` without a comment), test coverage against
  section 7, and plan drift. The coordinator, not the implementer,
  decides on deviations.
- Anything that needs the owner (enabling Actions, branch protection,
  a real cluster for M3 verification, signing decisions) is raised
  immediately in the PR and to the owner, and the implementer continues
  with everything that does not depend on it.

## 12. Risks

- Preview `xcode-27` runners may be slow or unavailable. Mitigated by
  keeping that job non-blocking.
- Stencil is unmaintained. Isolated behind a protocol; a replacement is
  a v3 item.
- Exec credential plugins may prompt interactively (browser SSO,
  `kubelogin`). The proxy will sit in `starting` until the prompt
  completes; the status menu shows retained stderr so the user can see
  why. Document this.
- `kubectl proxy` stdout contract. Guarded by tests and by the fallback
  of probing the configured port if the line never appears within 10 s
  when `port != 0`.
- Hand-written pbxproj in M2 can only be validated in CI. Keep that PR
  focused and expect one or two CI iterations.

## 13. Sources consulted

- GitHub changelog: macos-26 GA for hosted runners (2026-02-26);
  `macos-latest` moving to macos-26 (runner-images issue #14167);
  Xcode 27 image public preview (2026-07-16).
- actions/runner-images `macos-26-arm64-Readme.md`: macOS 26.6.2,
  default Xcode 26.6, SwiftFormat preinstalled, no kubectl/xcodegen.
- Xcode 26.6 bundles Swift 6.3.3; swift.org lists Swift 6.4.0 for Linux.
- stencilproject/Stencil releases: 0.15.1 (2022-07-31) is latest.
- prometheus/alertmanager releases: 0.34.1 (2026-09-17).
- kubernetes.io: `kubectl proxy` reference and "Access services running
  on clusters"; `kubectl/pkg/cmd/proxy/proxy.go` for the
  `Starting to serve on %s` stdout message.
- GitHub docs: workflows on forks are disabled until enabled in the
  Actions tab.
