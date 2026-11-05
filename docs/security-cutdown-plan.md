# Plan ucięcia elementów wpływających na Security („Corporate build”)

Cel: wersja Orca, którą można uruchomić w sieci firmowej bez stresu — bez
ruchu do chmury Orca/stablyai, bez nasłuchu na interfejsach sieciowych, bez
mechanizmów zdalnego sterowania i bez cichego pobierania/uruchamiania kodu.
Lokalna praca (terminal, worktree, edytor, git, agenci CLI) zostaje.

## 1. Zasady

1. **Wycięcie w czasie builda, nie tylko w ustawieniach.** Ustawienie
   użytkownik może przełączyć, a audytor nie zweryfikuje go w binarce.
   Wzorzec już istnieje: `ORCA_POSTHOG_WRITE_KEY` / `ORCA_BUILD_IDENTITY`
   (`electron.vite.config.ts:36-49`) oraz `config/dark-build-switches.cjs`
   (stałe `define`, które minifikator wycina razem z martwymi gałęziami).
   Dodajemy jedną stałą `ORCA_CORPORATE_BUILD` i gałęzie `if (!ORCA_CORPORATE_BUILD)`
   w punktach startowych każdej funkcji.
2. **Domyślnie odmawiaj (default-deny egress).** W corporate buildzie główny
   proces dostaje allowlistę hostów (`src/main/network/http-client.ts`,
   `electron-proxy-request-guard.ts`); wszystko spoza niej (`*.onorca.dev`,
   `posthog`, `storage.googleapis.com`, …) jest blokowane i logowane.
3. **Tylko loopback.** Żaden listener nie może wyjść poza `127.0.0.1` /
   socket Unix / named pipe.
4. **Bezpieczne domyślne uprawnienia agentów** zamiast „full-access”.
5. **Weryfikowalność:** testy kontraktowe na zbudowanym artefakcie (grep po
   bundlu + test egress + skan portów), żeby regresja w upstreamie nie
   przywróciła wyciętej funkcji.

## 2. Inwentarz i decyzje

Priorytety: **P0** — usunąć przed pierwszym uruchomieniem w firmie,
**P1** — zmienić domyślne / zablokować polityką, **P2** — opcjonalnie, zależnie od polityki firmy.

### 2.1 Ruch wychodzący do usług Orca / stron trzecich (P0)

| Element | Gdzie | Co wysyła | Decyzja |
|---|---|---|---|
| Telemetria PostHog | `src/main/telemetry/*` (`client.ts`, `posthog-transport.ts`, `posthog-client-start.ts`, `consent.ts`) | zdarzenia użycia, install-id, kohorty eksperymentów | Usunąć: build bez `ORCA_POSTHOG_WRITE_KEY` (już daje `IS_OFFICIAL_BUILD=false`) **plus** twarde `ORCA_CORPORATE_BUILD` w `consent.ts` → zawsze `disabled`; usunąć zależność `posthog-node` z bundla. Telemetria renderera i `plugin-install-telemetry.ts` też. |
| Upload diagnostyki | `src/main/observability/diagnostic-bundle-upload.ts`, `diagnostic-upload-endpoint.ts`, `diagnostic-upload-http.ts` (`ORCA_DIAGNOSTICS_TOKEN_URL`) | logi, bundle diagnostyczny | Usunąć upload; zostawić tylko zapis lokalny (`local-file-sink.ts`) i „Zapisz do pliku”. |
| Feedback | `src/main/ipc/feedback.ts`, `feedback-request.ts` → `https://www.onorca.dev/v1/feedback` | treść zgłoszenia + diagnostyka | Usunąć IPC i UI. |
| Crash reporting | `src/main/crash-reporting/*` | `uploadToServer: false` już ustawione | Zostawić lokalnie; upewnić się, że `crash-feedback-diagnostic-bundle.ts` nie idzie przez feedback. |
| Star-nag | `src/main/star-nag/*` (`web-handoff.ts`, `prompt-session-telemetry.ts`) | otwieranie GitHuba, telemetria | Usunąć cały moduł. |
| Nudge / changelog | `src/main/updater/nudges/nudge-campaign.ts` (`onorca.dev/whats-new/nudge.json`), `updater/releases/updater-changelog.ts` | pobieranie zdalnego JSON | Usunąć. |
| Katalog pluginów | `src/main/plugins/catalog/plugin-kill-list-service.ts`, `plugin-install-stats-service.ts` (`onorca.dev/plugins/*.json`) | pobieranie list | Usunąć razem z marketplace (pkt 2.4). |
| Link previews | `src/main/link-preview/*`, ustawienie `nativeChatLinkPreviews: true` | pobiera dowolne URL-e z czatu (SSRF/wyciek do intranetu mimo `link-preview-address-guard.ts`) | Usunąć w corporate; default `false`. |
| Speech / dyktowanie | `src/main/speech/*` (pobieranie modeli, `openai-transcription-client.ts`) | audio do OpenAI, pobieranie modeli | Usunąć (P0, jeśli audio nie może opuszczać firmy). |
| Rate-limits / usage | `src/main/rate-limits/*`, `src/main/usage`, `*-usage` | odczyt tokenów OAuth agentów i odpytanie API dostawców | P1: wyłączyć domyślnie; nie czytać cudzych plików z poświadczeniami. |

### 2.2 Chmura Orca, relay i zdalny dostęp (P0)

| Element | Gdzie | Ryzyko | Decyzja |
|---|---|---|---|
| Mobile companion + relay | `src/main/runtime/mobile-cloud-relay/`, `mobile-desktop-relay/`, `mobile-pairing-*.ts`, `delegated-mobile-devices.ts`, `src/shared/remote-runtime/relay/*`, `cloud/apps/relay`, `mobile/` | wychodzący WebSocket do `relay.onorca.dev` → **zdalny dostęp do terminali z zewnątrz, z pominięciem firewalla** | Usunąć w całości (RPC, UI parowania, przycisk `showMobileButton`, bundled mobile web bundle `bundled-mobile-web-bundle.ts`). |
| Push gateway | `src/main/runtime/push/*` → `push.onorca.dev` | metadane zdarzeń wychodzą do chmury | Usunąć z relay. |
| Konta Orca Cloud | `src/main/orca-profiles/profile-cloud-*` (`login.onorca.dev`, PKCE na loopback) | logowanie do zewnętrznego IdP, synchronizacja org | Usunąć część „cloud”; profile lokalne zostają. |
| Connectors | `src/main/connector-cloud/*` → `connector.onorca.dev` (poll/outbox) | zewnętrzny kanał poleceń do maszyny | Usunąć (`experimentalConnectors`). |
| Artifacts sharing | `src/main/artifacts/artifact-cloud-*` → `share.onorca.dev` | publikacja treści na zewnątrz | Usunąć publikację; artefakty lokalne mogą zostać. |
| Skill sharing | `src/shared/skill-share-link.ts`, `src/main/skills/skill-client-mediated-transfer.ts`, `skill-bundle-*` | pobieranie/udostępnianie paczek skilli | Usunąć udostępnianie i instalację z linków. |
| Serwer LAN / runtime WS | `src/main/runtime/runtime-rpc/runtime-rpc-network-exposure.ts` (`WS_BIND_HOST_ALL_INTERFACES = '0.0.0.0'`), `rpc/ws-transport.ts`, `windows-firewall-remote-scope.ts` | nasłuch na wszystkich interfejsach po włączeniu parowania LAN | Przypiąć `pinnedBindHost = 127.0.0.1` zawsze (mechanizm już istnieje dla orcad sidecar, `main-process-runtime-launch.ts:107`); usunąć `ensureNetworkExposure` i reguły firewalla Windows. |
| `orca serve` / headless | `src/main/server/*`, `src/cli/handlers` (serve + pairing output) | wystawienie runtime do parowania | Usunąć tryb parowania; zostawić tylko lokalny socket CLI. |
| Ephemeral VMs | `src/main/ephemeral-vm/*` (`experimentalEphemeralVms`) | provisioning zdalnych maszyn | Usunąć. |

### 2.3 Auto-update i pobieranie kodu w trakcie działania (P0)

| Element | Gdzie | Decyzja |
|---|---|---|
| electron-updater | `src/main/updater/*`, `config/electron-builder.config.cjs:756` (`provider: 'github'`) | Wyłączyć całkowicie (`autoUpdater` nieładowany, `publish: null`). Aktualizacje dystrybuuje IT (MSI/pkg/deb) po własnym review. |
| Managed server / orcad na zdalnych hostach | `src/main/managed-server/*`, `src/main/ssh/orcad-*`, `managed-server/install/remote-node-runtime-host-download.ts`, `src/shared/node-runtime-pin.ts` | Wyłączyć automatyczne wgrywanie demona i Node na hosty SSH; P2: dopuścić tylko z wewnętrznego mirrora (`nodeDistMirrorUrl`) i paczek podpisanych przez firmę. |
| Pobieranie modeli speech | `src/main/speech/model-*` | Usunięte z 2.1. |

### 2.4 Rozszerzalność = wykonywanie obcego kodu (P0/P1)

| Element | Gdzie | Decyzja |
|---|---|---|
| System pluginów | `src/main/plugins/*`, `pluginSystemEnabled: true` | P0: `pluginSystemEnabled=false` wymuszone; usunąć marketplace/katalog, `devPluginPaths`. P2: allowlista pluginów zatwierdzonych przez IT (hash z `plugin-content-hash.ts`). |
| Rozszerzenia przeglądarki | `src/main/browser/extensions/*` (web-store, native-messaging) | Usunąć (`experimentalBrowserExtensions`), szczególnie native messaging. |
| Automations | `src/main/automations/*` | P1: wyłączyć domyślnie (cykliczne, nienadzorowane uruchamianie agentów). |
| Workspace hooks | `src/main/workspace-hooks/*`, `orca.yaml` setup scripts | P1: wymagać jawnego zaufania repo przed uruchomieniem skryptów (zostawić `agentWorkspaceTrustEnabled: true`, nie dać się wyłączyć). |

### 2.5 Dostęp do systemu i danych użytkownika (P0/P1)

| Element | Gdzie | Ryzyko | Decyzja |
|---|---|---|---|
| Computer use | `src/main/computer/*`, `native/computer-use-{linux,macos,windows}` | agent steruje myszą/klawiaturą i czyta ekran | P0: usunąć z buildu (moduły natywne i RPC). |
| Import haseł z przeglądarek | `src/main/browser-passwords/*` (`browser-password-chromium-import.ts`) | odczyt i deszyfrowanie bazy haseł Chrome/Edge | P0: usunąć import; P1: menedżer haseł wbudowanej przeglądarki wyłączony. |
| Czytanie danych innych aplikacji | `src/main/foreign-sqlite-readers/*` (Cursor), `src/main/ai-vault*`, `external-conversations`, `configuration-import`, `terminal-config-import` | skanowanie historii czatów/konfiguracji innych narzędzi | P1: wyłączyć domyślnie; import tylko na żądanie. |
| Zarządzanie kontami agentów | `claude-accounts`, `codex-accounts`, `cursor-accounts`, `grok-accounts`, `managed-data-accounts`, `rate-limits/claude-oauth-credentials.ts` | przechowywanie/przełączanie tokenów OAuth | P1: wyłączyć multi-konta; korzystać z natywnego logowania CLI. |
| Sekrety w ustawieniach | `opencodeSessionCookie`, `minimaxGroupId` w `default-global-settings.ts` | cookie sesji w pliku ustawień | P1: przenieść do `credentials/encrypted-api-key-file-store.ts` lub usunąć funkcję. |
| Emulator / adb | `src/main/emulator/*` (`mobileEmulatorEnabled: true`) | sterowanie urządzeniami | P1: wyłączyć. |
| Uprawnienia macOS (TCC) | `src/main/macos-permissions/*` | proszenie o Accessibility/Screen Recording | Zniknie razem z computer use; usunąć prośby. |
| Wbudowana przeglądarka | `src/main/browser/*` (CDP, `certificate-trust`, cookies, webauthn) | P1: zablokować ręczne akceptowanie niezaufanych certyfikatów (`certificate-trust-controller`), CDP tylko lokalnie, brak importu cookies. |

### 2.6 Domyślne ustawienia agentów i terminala (P1, ale szybkie)

W `src/shared/default-global-settings.ts`:

| Ustawienie | Teraz | Corporate |
|---|---|---|
| `agentPermissionDefault` | `'full-access'` (YOLO: `--dangerously-skip-permissions` itd., `tui-agent-permission-choice.ts:51`) | tryb z potwierdzeniami (`default`/ask); opcja full-access ukryta lub zablokowana polityką |
| `nativeChatPermissionMode` | `'full-access'` | tryb z potwierdzeniami |
| `nativeChatLinkPreviews` | `true` | `false` |
| `nativeChatInheritShellEnvironment` | `true` | P2: `false` (nie przekazywać całego env z sekretami) |
| `terminalAllowOsc52Clipboard` | `true` | `false` (program w terminalu/na zdalnym hoście może pisać do schowka) |
| `pluginSystemEnabled` | `true` | `false` |
| `browserClientHostedRemoteEnabled` | `true` | `false` |
| `mobileEmulatorEnabled`, `showMobileButton` | `true` | `false` |
| `agentWorkspaceTrustEnabled` | `true` | `true` + nieprzełączalne |
| `commitMessageAi.enabled` | `true` | P2: zgodnie z polityką dot. wysyłania diffów do LLM |

## 3. Mechanizm wdrożenia

1. **Stała builda** `ORCA_CORPORATE_BUILD` w `electron.vite.config.ts`
   (`define` dla main, preload, renderer, CLI), analogicznie do
   `ORCA_POSTHOG_WRITE_KEY_LITERAL`. Plik `src/shared/corporate-build.ts`
   eksportuje `isCorporateBuild` — jedno miejsce prawdy.
2. **Bramki w punktach rejestracji**, nie w głębi kodu: rejestracja IPC
   (`src/main/ipc`), RPC runtime (`src/main/runtime/rpc/methods`), start
   serwisów (`src/main/startup/*`), wpisy menu/UI (`src/renderer`).
   Dzięki temu bundler wytnie importy modułów i ich zależności
   (`posthog-node`, `electron-updater`, moduły natywne computer-use).
3. **Nadpisanie ustawień**: `getDefaultGlobalSettings` + warstwa
   „locked settings” stosowana przy odczycie ustawień (nie tylko przy
   pierwszym uruchomieniu — migracja istniejących profili).
4. **Allowlista egress** w `src/main/network/` dla `net.request`/`fetch`
   i sesji Electron (`session.webRequest.onBeforeRequest`); lista hostów z
   pliku polityki (np. `github.com`, wewnętrzny GitLab/Jira, proxy firmowe
   przez istniejące `httpProxyUrl`).
5. **Pakowanie**: `config/electron-builder.config.cjs` — `publish: null`,
   usunięcie `native/computer-use-*`, `native/windows-agent-hook` (jeśli
   niepotrzebny), mobile web bundle i modułów relay z `files`/`extraResources`.
   Własne podpisywanie kodu certyfikatem firmy.
6. **Opcjonalnie polityka zarządzana przez IT**: plik
   (`/etc/orca/policy.json`, `%ProgramData%\Orca\policy.json`,
   `/Library/Application Support/Orca/policy.json`) z allowlistą hostów i
   zablokowanymi ustawieniami — wspólna ścieżka dla mac/linux/windows.

## 4. Weryfikacja (warunek „bez stresu”)

1. **Test bundla**: skrypt w `config/scripts/` grepuje `out/` i paczkę
   instalacyjną — zero wystąpień `onorca.dev`, `posthog`, `relay`,
   `electron-updater`, `0.0.0.0`; brak plików `.node` computer-use.
2. **Test egress**: uruchomienie z `ORCA_BACKGROUND_LAUNCH=1` w kontenerze
   bez sieci poza allowlistą + logowanie prób połączeń (np. proxy
   rejestrujące) przez typowy scenariusz: start, otwarcie repo, worktree,
   agent, git push. Oczekiwane: tylko hosty z allowlisty.
3. **Skan portów**: `ss -ltnp` / `netstat -ano` w trakcie działania — tylko
   loopback i sockety lokalne (runtime, agent-hooks, daemon).
4. **SBOM + audyt zależności**: `pnpm check:dependencies`
   (`config/dependency-security.md`) na corporate buildzie; usunięte
   zależności znikają z SBOM.
5. **Regresja z upstreamu**: powyższe testy w CI dla każdego merge'a z
   `stablyai/orca`, żeby nowa funkcja sieciowa nie przeszła niezauważona.

## 5. Kolejność prac

1. Stała `ORCA_CORPORATE_BUILD` + `isCorporateBuild` + bezpieczne domyślne
   i zablokowane ustawienia (2.6) — najszybszy zysk.
2. Telemetria, feedback, upload diagnostyki, star-nag, nudge (2.1).
3. Relay/mobile/push/connectors/cloud profiles/artifact & skill sharing,
   bind tylko na loopback (2.2).
4. Auto-update i zdalna instalacja orcad (2.3).
5. Computer use, import haseł, pluginy, rozszerzenia przeglądarki (2.4–2.5).
6. Allowlista egress + testy weryfikacyjne w CI (3.4, 4).
7. Pakowanie, podpis, dokumentacja dla IT (lista hostów, porty, ścieżki danych).

## 6. Do decyzji po stronie firmy

- Czy agenci LLM (Claude Code, Codex itd.) w ogóle mogą wysyłać kod do
  dostawców — to poza Orca, ale Orca je uruchamia; ewentualnie ograniczyć
  `disabledTuiAgents` do zatwierdzonych.
- Czy SSH do zdalnych hostów jest dozwolone (zostawić bez auto-deploy
  orcad czy wyciąć całość `src/main/ssh`).
- Czy integracje GitHub/GitLab/Jira/Linear/Azure DevOps/Bitbucket/Gitea
  mają działać — tylko wobec instancji firmowych z allowlisty.
- Czy wbudowana przeglądarka jest potrzebna.
