# Proposals

Designs for Helix's own features, where there is no GNOME contract to
follow. A proposal says what Helix should do and why. Once it is
accepted and built, the contracts it defines move into `contracts/`
(interface XML, settings, behavior notes), where they are tested like
the GNOME ones. The proposal stays as the record of the decision.

Each proposal is one Markdown file, `NNNN-short-name.md`, starting with:

- **Status**: Draft, Accepted, Implemented (with the version), Rejected
  or Superseded (by which).
- **Summary**: a few sentences.

Then the motivation, the design, the alternatives considered, and the
open questions.

| # | Proposal | Status |
|---|----------|--------|
| 0001 | [Helix extensions](0001-extensions.md) | Accepted (Phases 0–2, API 1.1 implemented) |
