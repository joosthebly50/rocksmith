# Rocksmith 2014 (Steam, appid 221680) - launch crash + Real Tone Cable niet gedetecteerd

## Status: OPEN - refund-beslissing teruggedraaid, community-troubleshooting via Reddit loopt

## Context

Oorspronkelijk probleem: de Hercules/Real Tone USB gitaarkabel wordt door Bazzite
(PipeWire/ALSA) prima herkend (`alsa_card.usb-Hercules_Rocksmith_USB_Guitar_Adapter-00`),
maar niet door Rocksmith zelf op het gain-check scherm ("It sounds like your cable
is unplugged. Connect a Rocksmith Real Tone Cable.").

Tijdens het testen bleek de eerder gebruikte executable een getorrentte/gekraakte
P2P-release te zijn (`~/Downloads/Rocksmith.2014...-P2P/Rocksmith 2014/Rocksmith2014.exe`).
Daar is niet mee getest of gefixed - geweigerd. Rocksmith 2014 Remastered is
legitiem aangeschaft en via Steam geinstalleerd (appid 221680, installdir
`Rocksmith2014`).

Bij het testen van deze legitieme installatie bleek de game eerst **helemaal niet
meer te starten** (apart probleem, zie deel 1), en na het oplossen daarvan bleek
de oorspronkelijke kabel-detectie nog steeds kapot (deel 2).

---

## Deel 1: Launch crash (SIGSYS)

### Symptoom

- Spel start via Steam of handmatig via Proton, maar sluit binnen ~2-3 seconden af.
- Geen enkel venster verschijnt.
- Steam content log: herhaalde cycli van "App Running" -> "Fully Installed" binnen enkele seconden.

### Diagnose

- `journalctl` + `systemd-coredump`: proces `Rocksmith2014.e` (het echte 32-bit
  `Rocksmith2014.exe` onder wine) crasht met **SIGSYS (signal 31)**.
- Reproduceerbaar op **Proton - Experimental**, **Proton 11.0**, en **GE-Proton11-1**.
- Core dump (`coredumpctl` + `eu-readelf -n`) toont:
  - `si_signo: 31, si_code: 1` -> `SYS_SECCOMP`: het proces is gedood door een
    seccomp-filter, niet door een gewone crash.
  - `orig_rax: 28` -> syscall nr. 28 op x86_64 = `madvise`.
- De wine-preloader wordt door een **seccomp-sandbox van de Steam Linux Runtime 4.0
  container** (appid 4183110) vermoord op een doodnormale `madvise`-aanroep, nog
  voor er een venster bestaat.
- `steam steam://validate/4183110`: runtime bestanden zijn intact, geen corruptie.
- Geen SELinux AVC-denials, geen `SystemCallFilter=` op de systemd unit die Steam
  start - blokkade komt puur van de Steam Runtime container zelf.

### Fix

Alle Proton/GE-Proton builds die **Steam Linux Runtime 4.0** (appid 4183110)
vereisen crashen (Proton 11.0, Experimental, GE-Proton11-1). Builds die de oudere
**Steam Linux Runtime 3.0 "Sniper"** (appid 1628350) vereisen crashen niet:
**Proton 10.0** en **GE-Proton10-34**.

Fix toegepast: Steam's `CompatToolMapping` voor appid 221680 in
`~/.local/share/Steam/config/config.vdf` omgezet naar `proton_10` (later ook
getest met `GE-Proton10-34`). Vereist Steam volledig afsluiten (`steam -shutdown`)
voor het editen, anders overschrijft de lopende client de wijziging.

Resultaat: game start en blijft draaien. Launch-crash **opgelost**.

---

## Deel 2: Real Tone Cable niet gedetecteerd (WASAPI capture enumeratie)

Met een werkend gestart spel bleek het oorspronkelijke probleem nog steeds te
bestaan: het gain-check scherm blijft zeggen "cable is unplugged", ongeacht de
kabel (getest met 2 verschillende Real Tone cables - dus geen hardware-defect).

### Onderzoek / uitgesloten oorzaken

- **Host-audio werkt perfect**: PipeWire/ALSA "default" capture-device pakt
  automatisch de Hercules-adapter (`alsa_input.usb-Hercules_Rocksmith_USB_Guitar_Adapter-00`),
  bevestigd met een directe `arecord` test (state ging naar RUNNING).
- **Niet Steam Input/HID**: `winedevice.exe` had wel `/dev/hidraw1` (de HID-interface
  van de adapter) open, en er was veel dinput-pollingactiviteit, maar de officiele
  Ubisoft PC-configuratiehandleiding (`RocksmithConfiguration.pdf`, bijgeleverd)
  bevestigt dat de kabel via de standaard Windows **Recording devices**
  (WaveIn/WASAPI) hoort te lopen - het HID-spoor was een rode haring.
  `SDL_GAMECONTROLLER_IGNORE_DEVICES=0x12ba/0x00ff` als launch-option toegevoegd:
  geen effect.
- **Game's eigen audio-engine**: `Rocksmith.ini` met `DumpAudioLog=1` gezet ->
  `audiodump.txt` toont dat de game **Wwise/PortAudio** gebruikt voor audio, en
  specifiek de functie `NSoundInputMgr_Win32::DetectInputDevices_PortAudio` voor
  input-detectie. Deze functie logt consistent:
  `no new input devices, stop` - PortAudio's WASAPI capture-enumeratie levert
  simpelweg **nul apparaten** op, in elke geteste configuratie.
- **Ini-instellingen (uit de officiele Ubisoft-handleiding) geprobeerd, geen
  effect op detectie**:
  - `ExclusiveMode=0` (i.p.v. default 1)
  - `Win32UltraLowLatencyMode=0` (i.p.v. default 1)
  - `ForceWDM=1` - wordt door de game correct gelezen (zichtbaar in audiodump),
    maar lost niks op.
  - `ForceDirectXSink=1` - **wordt door de game genegeerd** (audiodump blijft
    `INIT ForceDirectXSink 0` tonen, ongeacht de ini-waarde of combinatie met
    ForceWDM). Lijkt een dode/vestigiale instelling in de Remastered-versie.
- **Wine registry DeviceState**: het capture-endpoint voor de adapter bestaat wel
  in Wine's registry (`HKLM\Software\[Wow6432Node\]Microsoft\Windows\CurrentVersion\MMDevices\Audio\Capture\{5A8BC895-...}`,
  hardware-ID matcht `USB\VID_12BA&PID_00FF`), maar had `DeviceState=4`
  (NOTPRESENT) i.p.v. 1 (ACTIVE). Geforceerd naar 1 via `wine reg add` - geen
  effect op PortAudio's enumeratie (register-cache wordt kennelijk niet door
  PortAudio/WASAPI geraadpleegd, of er wordt live opnieuw bepaald).
- **ALSA vs PulseAudio wine-driver**: `HKCU\Software\Wine\Drivers\Audio` van
  `alsa` naar `pulse` gezet. Verschil gevonden: in **winecfg's Audio-tab** toont
  de "Voice input device" dropdown met winepulse.drv de volledige, genoemde
  apparatenlijst inclusief **"Rocksmith Guitar Adapter Mono"** (met winealsa.drv
  toont die dropdown alleen generieke "(System default)"/"In: default" zonder
  namen - alsa-driver enumereert daar dus duidelijk kapot). Ondanks dat winecfg
  het apparaat wel ziet met pulse-driver, blijft PortAudio's WASAPI-laag in de
  game zelf `no new input devices, stop` melden. Dit bewijst dat winecfg een
  andere (niet-WASAPI) enumeratieroute gebruikt dan de game, en dat het echte
  probleem in Wine's **WASAPI/mmdevapi capture-endpoint-enumeratie** zit
  (`IMMDeviceEnumerator::EnumAudioEndpoints(eCapture, ...)`), niet in de
  onderliggende alsa/pulse driver zelf.
- **WINEDEBUG tracing** (`+dsound,+mmdevapi,+wdmaud,+winmm,+alsa,+pulse`): in alle
  logs verschijnen alleen `render_*` (output) mmdevapi-calls, nooit een enkele
  `capture_*` call. De game/PortAudio probeert dus nooit succesvol een capture-
  client te openen, ongeacht driver.

### Root cause (bevestigd)

**Wine/Proton's WASAPI audio-capture-endpoint-enumeratie retourneert nul
apparaten voor deze USB-gitaaradapter, in elke geteste combinatie van Proton-
versie en audio-driver.** Dit is een Wine/Proton-niveau beperking, niet iets
dat via Rocksmith's ini-instellingen, registry-edits, of driver-keuze op te
lossen is binnen de beschikbare tijd.

### Geprobeerd maar niet succesvol

- Proton 10.0 (Sniper-runtime, geen crash) - detectie blijft kapot.
- GE-Proton11-1 - crasht (vereist Runtime 4.0, zie Deel 1).
- GE-Proton10-34 (nieuwere Wine-patches, Sniper-runtime) - **crasht ook** met
  SIGSYS bij het testen (andere/nieuwe crash-instantie, niet verder
  gediagnosticeerd wegens tijdgebrek en de beslissing om te stoppen).

### Beslissing (herzien)

Oorspronkelijke beslissing was **refunden** via Steam (zie `CHANGELOG.md`
voor tijdlijn). Game staat op 2026-07-18 echter nog steeds geinstalleerd
(appid 221680, prefix `compatdata/221680`, gepinned op GE-Proton10-34) -
refund is niet doorgezet of game is opnieuw aangeschaft. Gebruiker is
doorgegaan met troubleshooten via een Reddit-post (bugreport-PDF +
gestitchte PNG bijgevoegd) in r/rocksmith en r/linux_gaming-achtige
subs.

---

## Deel 3: Community-troubleshooting (Reddit, 2026-07-18)

### Suggestie van Reddit-gebruiker "flynnguy" (link:
`r/rocksmith` post "Guide: Get Rocksmith working on Steam Deck")

Kreeg het aan de praat op Linux Mint met:
1. `protontricks 221680 sound=alsa`
2. `protontricks 221680 winecfg` -> Audio-tab: output op Default, input op
   het Rocksmith-kabelapparaat
3. In `Rocksmith.ini`: `ExclusiveMode=0` en `Win32UltraLowLatencyMode=0`

### Verificatie tegen bestaande staat (2026-07-18)

- Stap 3 (ini) stond al zo ingesteld sinds de oorspronkelijke Deel 2-tests
  - geen wijziging, al bevestigd zonder effect.
- Stap 2 (input-device expliciet instellen): bleek functioneel al het
  geval - in `user.reg` stond `winepulse.drv`'s `DefaultInput` al op de
  GUID van exact het Hercules-adapter-device
  (`alsa_input.usb-Hercules_Rocksmith_USB_Guitar_Adapter-00.mono-fallback`).
  Dus zelfs met het device expliciet als default input geselecteerd (het
  scenario dat flynnguy's winecfg-stap bereikt), bleef PortAudio in-game
  nul devices zien - al vastgesteld in Deel 2.
- Stap 1 (driver op `alsa` i.p.v. `pulse`): dit was op dit moment het enige
  nog niet in de huidige sessie herteste onderdeel (wel eerder getest in
  Deel 2, maar niet in combinatie met bovenstaande). Driver via
  `protontricks 221680 sound=alsa` omgezet
  (`Software\Wine\Drivers\Audio` -> `alsa`, bevestigd in `user.reg`).
- Live in-game test (gain-check scherm) vereist fysieke aanwezigheid van
  gebruiker + kabel; hertest uitgevoerd door gebruiker zelf na deze
  wijziging. Resultaat: **zie vervolg in changelog/deze sectie zodra
  bekend**.

### Fysieke laag gecontroleerd (n.a.v. tweede Ubisoft-reactie)

Ubisoft support herhaalde op 2026-07-18 hun eerdere generieke
Windows-antwoord (verkeerde USB 2.0/3.0-poort, powered hub, driver
herinstalleren, Windows 10). Gecontroleerd met `lsusb -t`:

- Apparaat (`12ba:00ff`) hangt direct aan de root hub (Bus 001, Port 005),
  niet achter een externe hub met beperkt powerbudget.
- Onderhandelt op **12M (USB Full-Speed / 1.1)** - dit is de native
  snelheid van dit apparaat (oud Sony-gelicenseerd USB-audioapparaat),
  geen degradatie of enumeratiefout.
- Geen power- of poort-gerelateerde symptomen zichtbaar op OS-niveau; dit
  bevestigt nogmaals dat het probleem niet in USB-stroomvoorziening zit
  maar in Wine's WASAPI/mmdevapi capture-enumeratie (zie Deel 2 root
  cause). Poort-wissel blijft een gratis test maar wordt niet verwacht
  iets op te lossen.

## Deel 4: SIGSYS-crash terug na herinstallatie (2026-07-18)

### Symptoom

Na herinstallatie van het spel (opnieuw aangeschaft/geinstalleerd na de
eerdere refund-beslissing) start het spel weer niet: dezelfde SIGSYS-crash
als Deel 1. Gebruiker meldde dit aanvankelijk als "boot problems" - bleek
na doorvragen het spel te zijn, niet het OS (systeem zelf draaide al 4
dagen stabiel, geen failed units, actieve grafische sessie).

### Diagnose

`coredumpctl list` toont 3x SIGSYS (signal 31) rond 19:34-19:35 op
**GE-Proton10-34**, exact het patroon uit Deel 1 (`SYS_SECCOMP`/Steam
Linux Runtime). CompatToolMapping in `config.vdf` bleek te staan op
`"GE-Proton10-34"` in plaats van de oorspronkelijk geverifieerde fix
`"proton_10"` (officiele Proton 10.0) - vermoedelijk teruggezet door de
herinstallatie, of nooit teruggezet na het testen van GE-Proton10-34 in
Deel 2. GE-Proton10-34 was al eerder gezien als **niet 100% betrouwbaar**
(1 crash-instantie genoteerd op 2026-07-16), dus dit is consistent.

### Fix toegepast

1. `steam -shutdown`, gewacht tot proces volledig afgesloten (vereist
   voordat `config.vdf` bewerkt kan worden).
2. `CompatToolMapping` voor appid `221680` in
   `~/.local/share/Steam/config/config.vdf` teruggezet van
   `GE-Proton10-34` naar `proton_10` (officiele Proton 10.0 - de
   oorspronkelijk geverifieerde, stabiele fix uit Deel 1).
3. Gebruiker moet Steam opnieuw starten en spel opnieuw testen.

### Les voor de toekomst

Gebruik **officiele Proton 10.0** (`proton_10`), niet GE-Proton10-34, voor
appid 221680 - laatstgenoemde crasht intermitterend ondanks dezelfde
(Sniper/Runtime 3.0) basis. Controleer na elke herinstallatie/update van
het spel of de CompatToolMapping nog op `proton_10` staat, want Steam kan
dit resetten.

## Deel 5: Doorbraak - mono capture-devices worden gefilterd (2026-07-18)

### Context

Na de Proton-fix (Deel 4) start het spel weer. Kabeltest herhaald met wine
audio-driver op `alsa` (ingesteld in Deel 3). Gain-check scherm meldt nog
steeds geen kabel, maar `audiodump.txt` laat nu **wel** enumeratie-activiteit
zien - een duidelijke verandering t.o.v. Deel 2 (waar `no new input devices`
zonder ooit iets te vinden verscheen).

### Bevinding

`NSoundInputMgr_Win32::DetectInputDevices_PortAudio` vindt nu 2 devices:

- Id 0: "USB Audio - USB Audio #1" `[p:1996 v:0b05]`
- Id 1: "default"

Geverifieerd met `lsusb`: `0b05:1996` = **ASUSTek USB Audio** (onboard
moederbord-audiochip), niet de Rocksmith-kabel. `arecord -l` bevestigt de
Rocksmith-adapter is **card 0**, en die verschijnt nergens in de
PortAudio-lijst.

`/proc/asound/cardX/streamY` vergelijking:

| Device | Card | Capture-kanalen |
|---|---|---|
| Rocksmith USB Guitar Adapter | 0 | **1 (mono)** |
| ASUS USB Audio #1 | 3 | 2 (stereo) |

**Root cause verscherpt**: Wine's WASAPI/PortAudio ALSA-enumeratie lijkt
**mono capture-only USB Audio Class devices te filteren/overslaan** - alleen
stereo-capture-apparaten worden opgenomen in de device-lijst. De
Rocksmith-kabel is per ontwerp mono (1 gitaarkanaal), wat verklaart waarom
hij nooit verschijnt, ongeacht driver (alsa/pulse) of registry-instellingen.

Dit is scherper dan de eerdere Deel 2-conclusie ("nul devices, geen
verklaring") - nu is er een concrete, reproduceerbare hypothese.

### Vervolgstap: virtuele fake-stereo ALSA-route (getest, mislukt)

`~/.asoundrc` aangemaakt met een `type pipewire` PCM die het mono
capture-kanaal van de Hercules-adapter (via PipeWire-node
`alsa_input.usb-Hercules_Rocksmith_USB_Guitar_Adapter-00.mono-fallback`)
oppakt, plus een `type route`-PCM eromheen die kanaal 0 dupliceert naar 2
uitgangskanalen (`rocksmith_stereo`).

- **Op de host werkt dit perfect**: `arecord -D rocksmith_stereo` neemt
  correct 2-kanaals audio op, geverifieerd met `ffmpeg -af astats` - beide
  kanalen identiek en niet stil (echt signaal, geduplice
  erd van het mono-bronkanaal). Device verschijnt ook keurig in
  `arecord -L`.
- **In-game/Wine ziet het niet**: na herstart van het spel met kabel
  aangesloten toont `audiodump.txt` nog steeds precies dezelfde 2 devices
  als voorheen (ASUS-onboard + "default") - `rocksmith_stereo` verschijnt
  niet in de PortAudio-enumeratie.

**Vermoedelijke oorzaak**: het spel draait niet in de normale
gebruikersomgeving maar in de **Steam Linux Runtime "Sniper"-container**
(appid 1628350, oudere Ubuntu-gebaseerde sandbox met eigen gebundelde
ALSA-libraries). Die container heeft waarschijnlijk geen (recente genoeg)
`pipewire`-ALSA-plugin, en/of de container-ALSA-config chaint niet door
naar de host's `~/.asoundrc`. Het probleem zit dus op een dieper niveau dan
Wine alleen - het is een **sandbox/container-isolatieprobleem** boven op
het mono-filter-probleem.

Nog niet geprobeerd: device definieren in `/etc/asound.conf` (systeembreed
i.p.v. user-scoped) voor het geval de container die wel leest, of
uitzoeken welke ALSA-runtime de Sniper-container precies bundelt en of
daar uberhaupt een `pipewire`- of `pulse`-PCM-plugin in zit.

## Deel 6: Meter reageert op geen enkel audiosignaal - vermoedelijk UI/render-bug (2026-07-18)

### Context

Na Deel 5 (kernel-level ALSA-loopback als fake-stereo device) bleek het
spel de Loopback wel te detecteren en te openen (Id 11/12, Slot 2/3), maar
zonder reactie op het gain-check scherm. In plaats van verder te sleutelen
aan routing is de vraag omgedraaid: **reageert het scherm uberhaupt op
enig audiosignaal, ongeacht bron?**

### Tests (alle met het spel live op het gain-check scherm)

1. **Luide 1kHz testtoon** rechtstreeks in de Loopback-playback-kant
   (`speaker-test -D hw:Loopback,0,0`), terwijl het spel de
   overeenkomstige capture-kant (`pcm1c/sub0`) exclusief open en
   `RUNNING` had (bevestigd via `/proc/asound/card4/pcm1c/sub0/status`,
   `owner_pid` = het wine-proces). **Geen reactie.**
2. **Live, echte kabel** (tweede kabel getest: C-Media `0d8c:0134` "USB
   PnP Audio Device", ook mono, via de Corsair-toetsenbord-hub) -
   `wpctl status` toonde een **actieve** PipeWire-verbinding
   `ALSA plug-in [wine-preloader] input_MONO < Rocksmith USB Guitar
   Adapter:capture_MONO [active]`, en de systeem-default input-source
   stond al op de Hercules-adapter. Dus Slot 1 ("default") kreeg
   aantoonbaar live kabelsignaal. **Geen reactie.**
3. **Hard klappen/geluid direct bij de PC** (test van Slot 0, het
   ASUS-onboard-apparaat dat het spel als eerste device opent).
   **Geen reactie.**

### Conclusie

Drie onafhankelijke, bevestigd-werkende audio-paden (Slot 0 ASUS via hw,
Slot 1 "default" via PipeWire naar de echte kabel, Slot 2/3 Loopback via
kernel-ALSA) leverden **alle drie geen enkele meter-reactie** op, ondanks
bevestigd live datastromen op OS/ALSA/PipeWire-niveau. Dit sluit
audio-routing als oorzaak nagenoeg uit.

**Nieuwe hypothese**: het gain-check-scherm zelf (de meter-widget) leest
de audiolevels niet correct uit, of rendert de meter niet bij - een
UI-/engine-laag-bug binnen het spel onder Wine/Proton, losstaand van welk
apparaat feitelijk is gekoppeld. Mogelijk gerelateerd aan eerdere
`GL_FRAMEBUFFER_INCOMPLETE_ATTACHMENT`-waarschuwingen gezien in
`kwin_wayland`-logs (niet bevestigd, wel een aanwijzing om te onderzoeken).

### Extra bevestiging: probleem is systeembreed binnen het spel (2026-07-18, later die avond)

Calibratiescherm heeft namelijk wel een "Disconnected"-modus waarmee je
alsnog het hoofdmenu kunt bereiken. Vandaar verder:

- **Tools > Options**: toont pas een "AUDIO SETTINGS"-item nadat `CTRL`
  ("Path/Input") is ingedrukt op het gewone Options-scherm - niet
  vanzelfsprekend zichtbaar.
- **Tools > Tuner**: toont exact dezelfde melding ("It sounds like your
  cable is unplugged. Connect a Rocksmith Real Tone Cable.") als het
  originele gain-check-scherm, volledig los van de eerste-launch-gate.

Dit bevestigt definitief dat het probleem **niet** samenhangt met het
first-launch-calibratiescherm specifiek, maar met de audio-input-detectie/
-weergave van het spel als geheel - elk scherm dat een levels-meter
gebruikt (calibratie, tuner, vermoedelijk ook mixer/audio settings) faalt
op dezelfde manier, ongeacht welk apparaat feitelijk is gekoppeld.

### WINEDEBUG-tracing (Deel 6, vervolg): audio komt correct binnen, bug zit binnen de game-exe

Steam launch options tijdelijk uitgebreid met
`WINEDEBUG=+wasapi,+mmdevapi,+alsa PROTON_LOG=1 PROTON_LOG_DIR=...` (Steam
moest voor het aanpassen van `localconfig.vdf` volledig afgesloten
worden). Resultaat: tientallen MB's grote trace-log, tijdelijk bewaard in
de scratchpad-directory van de Claude-sessie (niet in git vanwege
bestandsgrootte - bij interesse opnieuw te reproduceren via dezelfde
launch options).

Bevindingen (in tegenstelling tot de oorspronkelijke Deel 2-trace, die
**nooit** een `capture_*`-call zag):

- **183.549** `capture_*`-mmdevapi-calls in de log - `capture_GetBuffer`/
  `capture_ReleaseBuffer` cyclen continu en normaal.
- `alsa:alsa_create_stream` + `alsa:map_channels need_remapping: 0,
  alsa_channels: 1` - Wine opent daadwerkelijk het **echte mono
  ALSA-device** en leest er continu uit (`alsa_read_data read 144 from
  656 limit 1392`, etc.) - dit zijn echte, doorlopende reads, geen
  stilte/mock-data.
- **Geen enkele** `AUDCLNT_BUFFERFLAGS_SILENT`-flag, geen `err:`/`fixme:`
  regel voor mmdevapi behalve een onschuldige
  `AEV_SetMasterVolumeLevelScalar stub`.
- Property-store-queries (friendly name, audio-engine-deviceformat,
  endpoint-formfactor) worden allemaal beantwoord zonder fouten.

**Conclusie**: Wine's WASAPI/ALSA-implementatie doet exact wat hij moet
doen - open device, lever echte audio-samples, beantwoord metadata-
queries. Er is op dit niveau **niets fout te vinden**. De resterende bug
zit dus onvermijdelijk **binnen de closed-source Rocksmith2014.exe**
zelf - vermoedelijk een subtiel verschil in WASAPI-timing/padding/
positie-semantiek tussen Wine's implementatie en echte Windows, waar de
game's eigen signaal-analyse-code niet tegen bestand is. Dit is met
extern ALSA/PipeWire-gesleutel (dit hele onderzoek) niet verder op te
lossen - vereist ofwel een Wine-bugreport (met deze trace als bewijs),
ofwel reverse engineering van de game-exe zelf.

Launch options na dit onderzoek **niet automatisch teruggezet** - check
bij een volgende sessie of `WINEDEBUG`/`PROTON_LOG` nog aanstaan in Steam
(properties > launch options voor Rocksmith 2014) en verwijder ze indien
niet meer nodig voor prestaties.

### Nog niet onderzocht

- Of het gain-check-scherm uberhaupt de juiste audio-analyse-callback
  aanroept onder Wine (zou dieper Wine-debug-tracing of reverse
  engineering vereisen).
- Of overschakelen van Wayland naar X11/Xwayland voor de game-sessie iets
  uitmaakt (i.v.m. de framebuffer-waarschuwingen).
- Of dit ook optreedt op een compleet schone/opnieuw aangemaakte Proton-
  prefix (huidige prefix heeft veel historie van eerdere driver/registry-
  experimenten - niet uit te sluiten dat een prefix-reset iets verandert,
  al is dat qua bewijslast onwaarschijnlijk gezien de consistentie van
  bovenstaande resultaten).

### Als iemand dit later oppakt

- Focus op Wine's mmdevapi source (`dlls/mmdevapi/`) capture-endpoint
  enumeratie-pad specifiek voor USB Audio Class devices met 1 kanaal (mono).
- Vergelijk met een recentere/oudere Wine-versie los van Proton/GE-Proton
  (bv. system-wine via distrobox) om te zien of dit Proton-specifiek is of
  een generieke Wine-bug.
- Overweeg een Wine-bugreport met de exacte hardware-ID
  (`USB\VID_12BA&PID_00FF`, "Hercules Rocksmith USB Guitar Adapter") en de
  bevinding dat winecfg het apparaat wel ziet (met pulse-driver) maar
  WASAPI/mmdevapi's `EnumAudioEndpoints(eCapture)` niet.
