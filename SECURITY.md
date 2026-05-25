# Security Policy

## Scope

This repository documents local security research on a WordPress REST API authorization boundary issue involving featured_media.

The research was performed in a controlled local laboratory environment using test users, test posts, and test media only.

## Responsible disclosure status

The issue was submitted through the appropriate coordinated disclosure channel before public documentation.

This repository is intended as a technical writeup and reproducibility record. It is not intended to encourage attacks against third-party WordPress installations.

## Safe use

Use the material only on systems you own or where you have explicit authorization.

Do not test against third-party WordPress sites without permission.

Do not use the PoC material to access, expose, or publish data belonging to other people.

## Impact summary

The documented issue shows that a lower-privileged authenticated user can reference a private administrator-owned attachment as featured_media on a post they control, despite not being allowed to read that media directly through the REST API.

Depending on role and workflow, this can lead to public exposure of the media file and related rendered metadata such as alt text.

## Contact

For questions about this research, open a GitHub issue on this repository.
