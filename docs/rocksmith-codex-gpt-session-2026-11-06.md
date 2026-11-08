# Rocksmith — Codex/GPT sessie

Bijgewerkt: 2026-11-06

## Doel

De extreme ruis en te hoge gitaar-gain in Rocksmith 2014 op Bazzite oplossen zonder de werkende Real Tone-verbinding te breken.

## Werkwijze

- GPT interpreteert bevindingen en geeft één volgende gecontroleerde stap.
- Codex voert die stap uit, rapporteert het resultaat terug en controleert GPT opnieuw.
- Per test wordt één variabele gewijzigd; wijzigingen zijn tijdelijk en krijgen rollback.
- Geen globale PipeWire-, Wine- of desktopwijzigingen zonder expliciete opdracht.

## Vastgestelde omgeving

- Bazzite KDE/Fedora Atomic, Wayland
- Real Tone Cable USB `12ba:00ff`
- ALSA card 0, device 0, mono, S16_LE, 48 kHz
- Werkende Protontricks-route: Proton Experimental, prefix/AppID `2661300`
- Rocksmith PID tijdens live tests: `21276`
- Actieve executable vanuit `~/Downloads/Rocksmith.2014/Rocksmith 2014/`

## Belangrijkste bevindingen

## Werkende Real Tone-kabelconfiguratie

De kabel werkt in de huidige setup via deze combinatie:

- Bazzite KDE / Fedora Atomic / Wayland
- Real Tone Cable via USB `12ba:00ff`
- ALSA capture: card 0, device 0, mono, S16_LE, 48 kHz
- Rocksmith gestart via de bestaande Protontricks Launcher-methode
- Proton Experimental in het eerder werkende prefix/AppID `2661300`
- De actieve Rocksmith-kopie wordt vanuit de bestaande Downloads-locatie gestart
- De kabel wordt door Rocksmith rechtstreeks als ALSA-captureapparaat geopend

Deze combinatie moet behouden blijven. Niet overstappen naar AppID `221680`, een ander prefix of een nieuwe launchmethode. Deze configuratie verklaart waarom de kabel wordt herkend en input levert; de precieze oorzaak van de te hoge hoorbare gain is hiermee nog niet definitief opgelost.

## Volledige uitgevoerde stappen

1. De Real Tone-kabel via USB aangesloten en gecontroleerd met USB-ID `12ba:00ff`.
2. ALSA gecontroleerd met `cat /proc/asound/cards` en `arecord -l`; de kabel verscheen als card 0/device 0 (`USB Audio`).
3. PipeWire-bronnen, sinks, source-outputs en sink-inputs read-only geïnventariseerd.
4. Vastgesteld dat de Real Tone-bron `alsa_input.usb-Hercules_Rocksmith_USB_Guitar_Adapter-00.mono-fallback` is.
5. Vastgesteld dat de systeem-defaultbron aanvankelijk de Turtle Beach Stealth Pro Xbox was; daarom is de defaultbron niet globaal gewijzigd.
6. De werkende Protontricks Launcher-route en het bestaande prefix/AppID `2661300` behouden.
7. Rocksmith gestart en gecontroleerd dat de kabel door de game wordt geopend.
8. De actieve Rocksmith-PID, Wine/Proton-processen, prefixomgeving en audio-backend geïnspecteerd zonder credentials te publiceren.
9. Vastgesteld dat Rocksmith het ALSA-capturedevice rechtstreeks bezet (`/dev/snd/pcmC0D0c`).
10. Het Rocksmith-venster tijdelijk zichtbaar gemaakt toen het buiten het bureaublad stond; dit wijzigde geen audio- of kabelinstelling.
11. Een tijdelijke PipeWire-mute-test uitgevoerd en automatisch teruggedraaid; de hoorbare ruis veranderde niet.
12. De ALSA-control `Mic Capture Volume` read-only vastgesteld: bereik 0–37, oorspronkelijke waarde 37 (+10 dB).
13. Een tijdelijke A/B-test met een lagere Mic-waarde uitgevoerd en automatisch hersteld naar 37. De luisterwaarneming was tegenstrijdig en leverde geen betrouwbare oorzaak op.
14. Geconstateerd dat de open instrumentplug al veel ruis geeft voordat de gitaar wordt aangesloten.
15. Geconstateerd dat Rocksmith-audio wegvalt bij minimaliseren of focusverlies; dit is als afzonderlijk window/audio-probleem vastgelegd.
16. Geen permanente gain-, PipeWire-, Wine- of ALSA-wijziging doorgevoerd, omdat de bron van de gain nog niet eenduidig is bewezen.

1. Rocksmith bezit rechtstreeks `/dev/snd/pcmC0D0c`; `fuser` toont PID 21276.
2. ALSA capture is actief: mono S16_LE, 48 kHz, period 256, buffer 1024.
3. De enige directe hardware-control is `Mic Capture Volume`:
   - bereik 0–37
   - huidige waarde 37 = 100% = +10 dB
   - capture switch staat aan
4. Wine/PipeWire heeft daarnaast een capture-stream `4070` die naar Xbox-source 84 wijst. Deze stream mute'en veranderde de hoorbare ruis niet.
5. De open instrumentplug (kabel in USB, nog niet in de gitaar) veroorzaakt al zeer harde hoorbare ruis.
6. Rocksmith-audio verdwijnt wanneer focus verloren gaat of het venster wordt geminimaliseerd. Het venster stond eerder buiten beeld op `y=2290`; het is tijdelijk naar een zichtbare positie verplaatst.
7. `Rocksmith.ini` van de actieve Downloads-kopie bevat onder meer:
   - `ExclusiveMode=0`
   - `Win32UltraLowLatencyMode=0`
   - `Fullscreen=0`
   - `LatencyBuffer=4`

## Uitgevoerde tests

- Read-only PipeWire-, ALSA-, Wine/Proton- en vensterinspecties.
- Tijdelijke mute van source-output 4070: geen hoorbaar verschil.
- ALSA A/B-test `Mic 37 -> 20 -> 37`, met automatische restore. De gebruiker meldde dat het geluid tijdens de test juist harder werd. Omdat deze test aanvankelijk niet met Rocksmith zichtbaar/gefocused was uitgevoerd, is de eerste luisterwaarneming ongeldig verklaard.
- Herhaalde gefocuste A/B-test met Rocksmith geactiveerd; waarde is na afloop teruggezet naar 37. De gebruiker meldde opnieuw dat het geluid harder werd.

## Huidige veilige toestand

- ALSA `Mic` staat hersteld op 37 (+10 dB).
- Geen permanente PipeWire-, Wine- of Rocksmith-configuratie gewijzigd.
- Real Tone Cable wordt nog steeds door Rocksmith gedetecteerd en geopend.
- De precieze oorzaak van de hoorbare gain is nog niet definitief vastgesteld.

## Openstaande volgende stap

GPT moet de tegenstrijdige A/B-waarneming interpreteren en één volgende meetstap geven. Eerst moet worden onderscheiden of de hoorbare ruis uit de directe Real Tone ALSA-route, Rocksmith-inputverwerking of een andere playbackroute komt. Pas daarna mag een blijvende gain-aanpassing worden overwogen.

## Hervatten op de volgende sessiedag

Lees dit document eerst volledig. Controleer daarna read-only:

1. Of Rocksmith nog draait en welke PID/window-ID actief is.
2. Of `/dev/snd/pcmC0D0c` nog door Rocksmith wordt beheerd.
3. Of ALSA `Mic` nog op 37 staat.
4. Of de Real Tone-USB-adapter nog `12ba:00ff` is.

Stuur deze status eerst naar GPT. Begin geen nieuwe gain-test voordat GPT de tegenstrijdige melding “Mic 20 werd harder” heeft geïnterpreteerd. Behoud de bestaande Protontricks-route en gebruik geen AppID 221680.

## Laatste gebruikersobservatie

Rocksmith-audio valt weg zodra de game wordt geminimaliseerd of zodra een ander venster wordt aangeklikt. Dit moet in de volgende sessie samen met GPT verder worden opgelost; behandel het als een afzonderlijk focus/audio-outputprobleem naast de hoge Real Tone-gain.

## Bewaarlocaties

- Diagnostische logs: `~/rocksmith-audio-debug/`
- Dit sessiedocument: `rocksmith-codex-gpt-session.md`

## 2026-11-08 — Eindstatus

De officiële Steam-installatie gebruikt AppID `221680` met Proton 10. De Real Tone Cable (`12ba:00ff`) werkt via ALSA card 0/device 0 op mono 48 kHz. `RealToneCableOnly=1` blijft actief zodat de Turtle Beach Stealth Pro-microfoon apart blijft.

RSMods is geïnstalleerd. De lokale `xinput1_3.dll` en `d3dx9_42.dll` zijn geladen; `AllowAudioInBackground=on` is ingesteld. Geminimaliseerd geluid werkte tijdens een succesvolle test, maar moet na nieuwe herstarts opnieuw worden gevalideerd.

Custom Forge CDLC is hersteld met CustomsForge CDLC Enabler v4.0. Na herstart meldde de game `Enumerating downloadable content`; de gebruiker bevestigde dat de DLC terug is. De officiële `dlc/`-map bevat 640 `.psarc`-bestanden.

Gain is nog niet volledig opgelost. Ruwe opname: RMS `-28.46 dBFS`, piek `-4.1 dBFS`, zonder clipping. ALSA `Mic=8/37` (`-4.11 dB`) klinkt duidelijk beter dan 37/37; 2/37 werd nog zachter.

PipeASIO/PipeWire is de volgende geplande test. Herstelpunten staan in `pre-pipeasio-20261108-153703` en `rocksmith-complete-20261108-144556`; de complete backup is ook gekopieerd naar `~/Homelab/Rocksmith/` en `~/Documents/Rocksmith/`.
