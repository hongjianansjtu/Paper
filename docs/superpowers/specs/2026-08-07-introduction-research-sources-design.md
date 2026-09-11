# Introduction Research Sources Design

## Goal

Create a reviewable evidence pack for the Introduction of the cross-chain atomic-swap paper. The pack explains what the supervisor's annotation is asking for and records quantitative comparisons, real deployments, market context, source quality, and remaining evidence gaps.

## Scope

- Add `sources/introduction-research.md` as the only research-content file.
- Keep the paper source files and `refs.bib` unchanged.
- Use a fixed 2026-08 snapshot, state the data source and query, and avoid mixing incompatible market-cap taxonomies.
- Classify sources into primary/official, reproducible data, and auxiliary/lead-only evidence.

## Structure

1. Translate the annotation into Introduction claims and evidence needs.
2. Define terminology and comparison axes, especially the difference between public-chain per-transaction fees and consortium-chain infrastructure costs.
3. Give reproducible Coin Metrics fee and market-cap snapshots.
4. Record protocol-level privacy/scalability evidence and production projects.
5. List optional bridge/regulatory motivation, risks, and claims that should not be used without stronger verification.
6. End with a short checklist and candidate prose for the Introduction.

## Non-goals

- No automatic insertion of claims into `paper.tex`.
- No new BibTeX entries in this branch.
- No claim that Haveno is an atomic-swap implementation; it is listed separately as a multisig/arbitration P2P DEX.
