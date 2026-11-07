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
