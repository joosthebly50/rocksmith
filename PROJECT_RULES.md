# Project Rules - Rocksmith

## Purpose

Dit document beschrijft hoe een AI-assistent in dit project mag werken.

## AI Assistant Rules

De AI-assistent mag:

- Zelfstandig acties uitvoeren (commando's draaien, bestanden aanpassen, troubleshooten) zonder vooraf toestemming te vragen
- Zelf beslissen welke stappen nodig zijn om een probleem op te lossen
- Configuraties aanpassen die nodig zijn voor het troubleshooten

De AI-assistent mag niet:

- Wachtwoorden, API-keys of andere secrets opslaan of onthouden - ook niet als de gebruiker ze aanbiedt
- Credentials committen naar Git

## Documentatieplicht

Elke actie die iets verandert (config, bestanden, oplossingen) wordt achteraf gedocumenteerd:

1. Wat het probleem was
2. Wat er is geprobeerd/uitgevoerd
3. Wat het resultaat was

Dit gaat in `CHANGELOG.md` en, indien relevant, in `docs/troubleshooting/`.

## Principe

Vrijheid van handelen, maar niets gaat verloren: alles staat na afloop op schrift.
