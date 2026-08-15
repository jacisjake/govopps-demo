# GovOpps demo spec

Public contract for **GovOpps** — a GOED / StartupState studio that translates a Utah startup into government language, maps federal and Utah opportunities, and opens a checklist toward a submission.

Copilot persona: **McKenna**.

This repository is the **rules**. The running product (API, scorer implementation, dashboard) lives in a private repo. If the two ever disagree, **this spec wins** for product behavior; `scoring_schema.md` is the sole scoring authority.

## Start here

- [spec/README.md](spec/README.md) — index
- [spec/architecture.md](spec/architecture.md) — pipeline
- [spec/harness.md](spec/harness.md) — facet ticket, compiler, eval
- [spec/scoring_schema.md](spec/scoring_schema.md) — gates, tiers, record contract
- [docs/harness-c.html](docs/harness-c.html) — illustrated harness note
- [mock_startup/](mock_startup/) — five brief companies as scrape-able HTML

Hackathon brief: [Government Opportunity Finder](https://startupstate-hackathon-brief.lovable.app/).

Updates are published automatically when the private product repo changes `spec/`, `docs/`, or `mock_startup/`.

## License

Hackathon work. Not a production eligibility determination.
