# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in GTM Wizard, please report it privately.

**Email:** corneliu@dvision.cl

Please include:

- Description of the vulnerability
- Steps to reproduce
- Potential impact

We will acknowledge your report within 48 hours and provide an update within 7 days.

**Do not open a public issue for security vulnerabilities.**

## Scope

GTM Wizard processes GTM container export files locally. Security concerns may include:

- Scaffold templates that could produce unsafe tag configurations
- Research agent queries that could leak container details
- Patch generator output that could introduce unintended changes
