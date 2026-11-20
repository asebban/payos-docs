# Architecture — Validation de licence (obsolète — voir v1)

**Dernière mise à jour :** 2026-10-31

Ce document décrivait un mécanisme qui n'existe plus : `BootServer` câblant `DevLicenseValidator` en dur, et un `LicenseValidator` ne vérifiant que l'existence du fichier pointé par `license-file-path`. Depuis le travail de 2026-10, la sélection se fait par variable d'environnement, `LicenseValidator` effectue une validation cryptographique complète, et `license-file-path` n'est plus lu.

**Voir à la place : [license-validation-architecture-v5-2026-11-20.md](license-validation-architecture-v5-2026-11-20.md).**
