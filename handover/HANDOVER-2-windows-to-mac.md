# Handover 2: Windows-sessie → Mac (2026-09-14)

Vervolg op `HANDOVER.md` (Mac → Windows, 2026-09-12). Dat document blijft
relevant voor de achtergrond van het relais-klik-probleem en het
hardware-experiment; dit document beschrijft waar we nu staan en wat er nog
open staat, met als nieuw hoofdonderwerp: resterende meet-hikjes.

## Wat is klaar

- Fix omgedoopt van `keepLevelsOnStop` naar **`resetLevelsOnClose`**
  (default `true` = huidig/ongewijzigd gedrag, `false` = niveaus behouden
  bij stop). QA401 reset **altijd**, ongeacht de optie — dat is een
  bugfix voor een lingering DC-offset, geen keuze. Alleen QA403/QA402
  luisteren naar de optie.
- Géén destructor/thread-vertragingsmechanisme meer (is geprobeerd,
  weer teruggedraaid op verzoek — te veel machinerie voor het probleem;
  zie git-historie van deze sessie als je de afweging terug wilt lezen).
- Twee **losse PR's** open vanaf `fleegstra/ASIO401` naar
  `dechamps/ASIO401`:
  - `reset-levels-on-close` — de feature zelf.
  - `fix-ci-windows-latest` — 3 losse commits om de GitHub Actions CI
    weer werkend te krijgen op `windows-latest` (pre-existing breakage,
    niks met de feature te maken): `CMAKE_POLICY_VERSION_MINIMUM=3.5`
    (libsndfile's oude `cmake_minimum_required` knapt op CMake 4),
    `sigstore/gh-action-sigstore-python` v2.1.0→v3.5.0 en
    `softprops/action-gh-release` v1→v2 (beide trokken de gedepreceerde
    `actions/upload-artifact@v3` binnen / draaiden op een verouderde
    Node-runtime), en een glob-fix (`*.sigstore` → `*.sigstore.json`,
    v3.5.0 hernoemde de output-extensie).
- Draft release staat klaar op de fork (tag `asio401-2.0-resetonclose`)
  met x64+x86 installer + sigstore-attestatie — nog niet gepubliceerd,
  wacht op Frank's akkoord.
- **Let op, git-remote-conventie is omgekeerd tussen de machines:**
  - Windows fysieke pc: `origin` = dechamps upstream, `fork` = fork.
  - Mac: `origin` = fork, `upstream` = dechamps (standaard GitHub-conventie).
- `.gitignore`: de `src/build/`-regel zit bewust **niet** in de PR's (was
  een ad-hoc lokale mapnaam-keuze, geen projectconventie) — desgewenst
  lokaal (ongecommit) toevoegen.
- GitHub-login op beide machines geregeld (Windows: Git Credential
  Manager; Mac: `gh auth login` of GCM, user's keuze).

## Open: resterende meet-hikjes in frequency response

Voorgeschiedenis (deze sessie): een eerdere hik bleek een externe
I/O-stall te zijn (ASIO401.log was tientallen MB's groot en groeide
synchroon op de audio-thread, mogelijk verergerd door Bitdefender
real-time scanning) — opgelost door logging uit te zetten. Met logging
uit + Steps' "I/O delay"/"mute transients" getuned, veel minder hikjes,
maar niet nul.

**Code-analyse (nog geldig, geen nieuwe bevindingen nodig):**
- Geen race tussen init/setup en de eerste read: `SetupDevice()` (incl.
  register-writes + interne 50ms sleep in `QA403::Reset()`) loopt
  synchroon vóór de read-loop begint.
- QA401 heeft een gedocumenteerde workaround voor "initiële garbage" na
  start (`initialInputGarbageInFrames` in `asio401.cpp`, i.v.m. bekend
  firmware-issue #5: QA401 speelt eerst oude samples terug, dan stilte).
  **QA403 heeft dit niet** — hardcoded op `0u`. Nooit apart getest of
  QA403 ook settling-tijd nodig heeft vlak na `Start()` (reg 8=5).
- Eerder losse testtool-idee (`qa403regtest.cpp`, raw register-writes via
  `QA40x`/`GetDevicesPaths`) is gemaakt maar nooit afgemaakt/gebruikt en
  weer verwijderd (Frank wilde 'm niet permanent in de build). Zou de
  basis kunnen zijn voor een tool die de eerste paar honderd ruwe
  ADC-samples na `Start()` wegschrijft, om te zien of er echt
  settling-troep in zit voor QA403 — vergelijkbaar met het originele
  relais-experiment (`handover/qa403_relay_test.cpp`, macOS/libusb).

**Nieuw vandaag:** zelfde soort klein hikje op de Parallels-VM (Windows
op ARM, op de Mac), zelfs met logging uit én I/O delay op 20ms (dubbel
van de 10ms die op de fysieke Windows-pc al hielp). Dat wijst eerder op
**extra/variabelere latency door de virtualisatielaag** (USB-passthrough
naar de VM, gedeelde ARM-cores met macOS) dan op een driverbug — maar dit
is niet hard geverifieerd, alleen een educated guess.

**Werkhypothese (nog te toetsen):** er bestaat geen universele
"theoretisch juiste" I/O-delay-waarde. Het is een empirische,
per-systeem round-trip-latency (ASIO-outputbuffer → DAC → DUT → ADC →
ASIO-inputbuffer → host-verwerking). Een VM met USB-passthrough zal
per definitie hoger én minder voorspelbaar zijn dan bare-metal. Voor
precisiemetingen is bare-metal (de fysieke Windows-pc) waarschijnlijk de
betere referentie; Parallels vooral geschikt voor development/bouwen,
niet per se voor de uiteindelijke meetvalidatie.

## Volgende stappen (voorstel voor de Mac-sessie)

1. Uitzoeken of QA403 een garbage-skip-fix nodig heeft zoals QA401 (het
   raw-sample-capture-testje hierboven).
2. Verder induiken in wat een realistische/juiste I/O-delay-instelling is
   en waarom, en of Parallels/ARM daar fundamenteel anders in gedraagt
   dan bare-metal.
3. PR's opvolgen (reacties van dechamps afwachten op beide PR's).
4. Draft release publiceren zodra Frank tevreden is met de inhoud.
