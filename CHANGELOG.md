# Changelog - Rocksmith

## 2026-07-15

- Project aangemaakt (`/var/home/Joost/rocksmith`)
- Documentatiestructuur opgezet: README.md, PROJECT_RULES.md, CLAUDE.md, docs/INDEX.md, docs/troubleshooting/, docs/decisions/
- Git repo geinitialiseerd
- Geweigerd te troubleshooten op de gevonden gekraakte P2P-installatie van Rocksmith 2014 (torrent in Downloads); pas verder gegaan nadat het spel legitiem via Steam is aangeschaft en geinstalleerd (appid 221680, 6.5GB)
- Rocksmith 2014 blijkt niet te starten: crasht met SIGSYS (seccomp/madvise-kill in Steam Linux Runtime 4.0 container), reproduceerbaar op Proton Experimental en Proton 11.0. Proton 10.0 crasht niet maar start ook geen zichtbaar spel. Zie `docs/troubleshooting/rocksmith2014-wont-launch-sigsys.md` - nog open, blokkeert de oorspronkelijke audio/kabeltest

## 2026-07-18

- Bugreport samengevat in `docs/rocksmith2014-linux-proton-bugreport.pdf` en gestitcht naar `docs/rocksmith2014-linux-proton-bugreport.png` voor een Reddit-post (kabel/WASAPI-probleem, r/rocksmith)
- Reddit-suggestie van gebruiker "flynnguy" (protontricks `sound=alsa`, winecfg input-device instellen, ini-settings) geverifieerd tegen bestaande Deel 2-bevindingen: 2 van de 3 stappen bleken al (functioneel) getest zonder effect; wine audio-driver alsnog omgezet naar `alsa` als enige nieuwe variabele, live hertest door gebruiker met kabel aangesloten
- Tweede reactie van Ubisoft support ontvangen (2026-07-18, ticket): herhaling van generiek Windows-advies (USB 2.0/3.0-poort, powered hub, driver herinstalleren, Windows 10 upgrade). USB-topologie gecontroleerd met `lsusb -t`: kabel hangt direct aan root hub, onderhandelt correct op native 12M full-speed - geen aanwijzing voor een power/poort-probleem, advies niet toepasbaar op deze Linux-setup
- Status van `docs/troubleshooting/rocksmith2014-wont-launch-sigsys.md` teruggezet van CLOSED/refund naar OPEN: game blijkt nog geinstalleerd, troubleshooting gaat door via community (Reddit)
- SIGSYS-launchcrash (Deel 1) weer opgetreden na herinstallatie van het spel: CompatToolMapping stond op `GE-Proton10-34` (intermitterend onstabiel) i.p.v. de geverifieerde `proton_10`. Steam afgesloten en `config.vdf` teruggezet naar `proton_10`; gebruiker moet Steam herstarten en opnieuw testen
- Doorbraak in kabelprobleem (Deel 5): met wine audio-driver op `alsa` enumereert PortAudio nu wel devices (voorheen nul), maar de Rocksmith-kabel (ALSA card 0, mono capture) wordt overgeslagen - alleen het stereo ASUS-onboard-apparaat (card 3) wordt gevonden. Sterke aanwijzing dat Wine's WASAPI/PortAudio-enumeratie mono-only USB-audiocapture-apparaten filtert
- Fake-stereo ALSA-workaround (`~/.asoundrc`, `type pipewire`+`type route`) gebouwd en op de host geverifieerd te werken, maar wordt niet gezien door het spel: waarschijnlijk omdat Rocksmith in de geisoleerde Steam Linux Runtime "Sniper"-container draait met eigen ALSA-libraries die geen toegang hebben tot de host's `~/.asoundrc`/pipewire-plugin. Zie Deel 5 in `docs/troubleshooting/rocksmith2014-wont-launch-sigsys.md`
- Kernel-level ALSA `snd-aloop` loopback opgezet (`sudo modprobe snd-aloop`) als fake-stereo bridge (ffmpeg dupliceert het mono kabelsignaal naar 2 kanalen) - dit apparaat wordt door het spel wel gedetecteerd en geopend (i.t.t. de `~/.asoundrc`-poging), maar het gain-check-scherm reageert nog steeds niet
- Doorslaggevende test (Deel 6): 3 onafhankelijke, bevestigd-werkende audio-paden getest (luide testtoon in Loopback, live echte kabel via PipeWire-default naar Slot 1, hard geluid bij ASUS-onboard Slot 0) - **geen enkele** gaf een meter-reactie. Dit wijst op een UI-/render-bug in het gain-check-scherm zelf onder Wine/Proton, niet op een audio-routingprobleem. Zie Deel 6 in `docs/troubleshooting/rocksmith2014-wont-launch-sigsys.md`
- Bevestigd dat probleem systeembreed is: ook Tuner en (na `CTRL`=Path/Input) Audio Settings tonen dezelfde "cable is unplugged"-melding, los van het eerste-launch-scherm
- WINEDEBUG-trace (`WINEDEBUG=+wasapi,+mmdevapi,+alsa PROTON_LOG=1`) opgezet via tijdelijke Steam launch-options-wijziging: toont dat Wine daadwerkelijk continu echte audio-samples leest van het mono ALSA-device, WASAPI capture/render-calls foutloos verlopen, geen silence-flags, geen mmdevapi-errors. Conclusie: de bug zit binnen de closed-source Rocksmith2014.exe zelf (vermoedelijk WASAPI-timing/padding-verschil t.o.v. Windows), niet meer op te lossen via host-side audio-configuratie. Zie Deel 6 (vervolg) in `docs/troubleshooting/rocksmith2014-wont-launch-sigsys.md`
- Uitgebreid, indienbaar Engelstalig technisch rapport geschreven: `docs/rocksmith2014-wine-wasapi-capture-bug-report.md` - bundelt de volledige onderzoekslijn (crash-fix, mono-filtering, Sniper-container-isolatie, loopback-workaround, WINEDEBUG-bewijs) voor eventuele indiening bij bugs.winehq.org. `docs/INDEX.md` bijgewerkt (dode link naar niet-bestaand `decisions/refund-rocksmith2014.md` verwijderd, status van de troubleshooting-case gecorrigeerd naar "open")
- Vervolg op Ubisoft-support-ticket #26368801 opgesteld (niet door mij verstuurd - gebruiker verstuurt zelf), met correctie dat de eerdere generieke Windows-/USB-antwoorden niet van toepassing zijn
- Markdown-naar-PDF-conversiepijplijn opgezet (Python `markdown` + `weasyprint`, script in scratchpad) omdat Ubisoft's ticketsysteem geen `.md`-bijlagen accepteert: `rocksmith2014-wine-wasapi-capture-bug-report.pdf`, `troubleshooting/rocksmith2014-wont-launch-sigsys.pdf` en `docs/CHANGELOG.pdf` gegenereerd
- Oorspronkelijke `docs/rocksmith2014-linux-proton-bugreport.md`/`.pdf` (2026-07-16, verouderd - noemde nog "refund") bijgewerkt met alle 2026-07-18-bevindingen (mono-filtering, Sniper-container-isolatie, loopback-workaround, 3-paden-test, WINEDEBUG-bewijs) en opnieuw gegenereerd
- `.png` (Reddit-post-afbeelding) opnieuw gestitcht vanuit de bijgewerkte PDF (11 pagina's, was 5)
- Nieuwe map `docs/troubleshooting/18-7-2026/` aangemaakt met alle bijlagen voor Ubisoft-ticket #26368801 (kopieen, originelen blijven op hun eigen plek staan): `rocksmith2014-wine-wasapi-capture-bug-report.pdf`, `rocksmith2014-wont-launch-sigsys.pdf`, `CHANGELOG.pdf`, `rocksmith2014-linux-proton-bugreport.pdf` + `.png`
- Voor de Reddit-update-post: eerst `rocksmith2014-linux-proton-bugreport.pdf` en `rocksmith2014-wine-wasapi-capture-bug-report.pdf` samengevoegd (ghostscript) tot een 20-pagina PDF en gestitcht tot `docs/rocksmith2014-full-report-everything.png` - bleek te lang; vervangen door `docs/rocksmith2014-today-18-7-2026.png`, gestitcht van alleen `rocksmith2014-wine-wasapi-capture-bug-report.pdf` (9 pagina's, uitsluitend het 2026-07-18-onderzoek)

## 2026-11-05 — Nieuwe kabeltest

Steam-installatie hersteld door gebruiker. Kabel 12ba:00ff zichtbaar. Nieuw audiodump ziet kabel maar openen faalt met -9996. Backup onder backups/. Eerste test: ExclusiveMode=1, Win32UltraLowLatencyMode=1, RealToneCableOnly=1. Resultaat nog te controleren. Virtuele inputdienst herstart; ydotool relatieve beweging werkt met flat profiel (factor 1.5).

## 2026-11-08 — Werkende Linux-audioketen en Custom Forge

- Officiële Steam-route: AppID `221680`, Proton 10, Real Tone Cable `12ba:00ff`, ALSA card 0/device 0, mono 48 kHz.
- RSMods `AllowAudioInBackground=on` en DLL-override `xinput1_3=n,b;d3dx9_42=n,b` toegepast; lokale DLL's zijn in het actieve proces geladen.
- CustomsForge CDLC Enabler v4.0 toegepast. Na herstart verscheen `Enumerating downloadable content` en Custom Forge-DLC werkte weer.
- Officiële `dlc/`-map bevat 640 `.psarc`-bestanden.
- `RealToneCableOnly=1` behouden; Stealth Pro blijft apart voor voice.
- Ruwe Real Tone-opname: RMS `-28.46 dBFS`, piek `-4.1 dBFS`, geen digitale clipping. Playback sample: RMS `-20.47 dBFS`, piek `-4.5 dBFS`, geen clipping.
- ALSA `Mic` getest op 37, 20, 2 en 8. De gebruiker vond 8/37 (`-4.11 dB`) duidelijk beter; dit is de huidige werkwaarde.
- Pre-PipeASIO-backup: `/var/home/Joost/rocksmith-audio-debug/backups/pre-pipeasio-20261108-153703`.
- Complete herstelbackup: `/var/home/Joost/rocksmith-audio-debug/backups/rocksmith-complete-20261108-144556`.
- Volgende test: PipeASIO/PipeWire vergelijken met directe ALSA, met rollback.
- GPT-6-terugkoppeling verwerkt: 37/37 (+10 dB) gaf hoorbare vervorming; 8/37 (-4.11 dB) klinkt duidelijk beter. Geen digitale clipping gemeten; analoge/inputvervorming blijft mogelijk.
- GPT-6-plan voor de volgende sessie: huidige configuratie veiligstellen, PipeASIO/PipeWire testen, latency/ruis/geluidskwaliteit objectief vergelijken met ALSA, Stealth Pro-mic apart houden, achtergrond-audio als afzonderlijk probleem testen en volledig kunnen terugrollen.
