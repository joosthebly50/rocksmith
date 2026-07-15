# Rocksmith 2014 (Steam, appid 221680) start niet - SIGSYS crash

## Context

Oorspronkelijk probleem: de Hercules/Real Tone USB gitaarkabel wordt door Bazzite
(PipeWire/ALSA) prima herkend (`alsa_card.usb-Hercules_Rocksmith_USB_Guitar_Adapter-00`),
maar niet door Rocksmith zelf op het gain-check scherm.

Tijdens het testen bleek de eerder gebruikte executable een getorrentte/gekraakte
P2P-release te zijn (`~/Downloads/Rocksmith.2014...-P2P/Rocksmith 2014/Rocksmith2014.exe`).
Daar is niet mee getest of gefixed. Rocksmith 2014 Remastered is legitiem aangeschaft
en via Steam geinstalleerd (appid 221680, installdir `Rocksmith2014`).

Bij het testen van deze legitieme installatie bleek de game **helemaal niet meer te
starten** - een nieuw, apart probleem los van de kabel-detectie.

## Symptoom

- Spel start via Steam of handmatig via Proton, maar sluit binnen ~2-3 seconden af.
- Geen enkel venster verschijnt.
- Steam content log: herhaalde cycli van "App Running" -> "Fully Installed" binnen enkele seconden.

## Diagnose

- `journalctl` + `systemd-coredump`: proces `Rocksmith2014.e` (het echte 32-bit
  `Rocksmith2014.exe` onder wine) crasht met **SIGSYS (signal 31)**.
- Reproduceerbaar op zowel **Proton - Experimental** als **Proton 11.0**.
- Core dump (`coredumpctl` + `eu-readelf -n`) toont:
  - `si_signo: 31, si_code: 1` -> `SYS_SECCOMP`: het proces is gedood door een
    seccomp-filter, niet door een gewone crash.
  - `orig_rax: 28` -> syscall nr. 28 op x86_64 = `madvise`.
- Dus: de wine-preloader wordt door een **seccomp-sandbox van de Steam Linux
  Runtime 4.0 container** (appid 4183110, build `steamrt4_platform_4.0.20260608.242786`)
  vermoord op een doodnormale `madvise`-aanroep, nog voor er een venster bestaat.
- `steam steam://validate/4183110` uitgevoerd: runtime bestanden zijn intact, dus
  geen corruptie.
- Geen SELinux AVC-denials rond het tijdstip van de crash (`ausearch -m avc`).
- Geen `SystemCallFilter=` op de systemd user-unit die Steam start
  (`app-steam@autostart.service`), dus de blokkade komt niet van systemd-hardening.

## Conclusie (voorlopig)

Vermoedelijke oorzaak: incompatibiliteit tussen de zeer recente Bazzite-kernel
(`7.1.3-ogc3.4.fc44`) en de seccomp-syscallfilter die de Steam Linux Runtime 4.0
container (pressure-vessel) toepast op het gesandboxte proces. Dit lijkt een
upstream bug/mismatch, niet iets dat lokaal met een simpele instelling op te
lossen is.

Getest als workaround: **Proton 10.0** i.p.v. 11.0/Experimental.
- Geen SIGSYS-crash meer.
- Maar: het spel start ook niet zichtbaar op (geen proces meer actief na
  wineboot-initialisatie, geen venster, geen foutmelding). Nader te onderzoeken
  of dit een Steam Runtime versie-mismatch is (Proton 10.0 verwacht mogelijk een
  oudere/andere runtime dan de geinstalleerde Runtime 4.0).

## Status

Nog niet opgelost. De game start momenteel niet betrouwbaar met geen van de
geteste Proton-versies, waardoor de oorspronkelijke audio/kabel-troubleshooting
nog niet eens bereikt kan worden (dat scherm komt pas na inloggen, wat een
werkend gestart spel vereist).

## Volgende stappen (nog niet uitgevoerd)

- Checken op Bazzite/kernel updates (`rpm-ostree status` / `ujust update`) -
  mogelijk is dit al gefixed in een nieuwere image.
- Proton 10.0 verder debuggen: waarom sluit het stil af na wineboot init.
- Overwegen: GE-Proton (community build) proberen als die niet dezelfde
  Runtime 4.0 seccomp-interactie heeft.
- Zoeken/melden bij Bazzite/ValveSoftware Proton issue tracker met deze exacte
  kernel + Steam Runtime 4.0 build combinatie.
