# Changelog - Rocksmith

## 2026-07-15

- Project aangemaakt (`/var/home/Joost/rocksmith`)
- Documentatiestructuur opgezet: README.md, PROJECT_RULES.md, CLAUDE.md, docs/INDEX.md, docs/troubleshooting/, docs/decisions/
- Git repo geinitialiseerd
- Geweigerd te troubleshooten op de gevonden gekraakte P2P-installatie van Rocksmith 2014 (torrent in Downloads); pas verder gegaan nadat het spel legitiem via Steam is aangeschaft en geinstalleerd (appid 221680, 6.5GB)
- Rocksmith 2014 blijkt niet te starten: crasht met SIGSYS (seccomp/madvise-kill in Steam Linux Runtime 4.0 container), reproduceerbaar op Proton Experimental en Proton 11.0. Proton 10.0 crasht niet maar start ook geen zichtbaar spel. Zie `docs/troubleshooting/rocksmith2014-wont-launch-sigsys.md` - nog open, blokkeert de oorspronkelijke audio/kabeltest
