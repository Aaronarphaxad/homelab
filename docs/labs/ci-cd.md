# CI/CD

## Status

Planned

## Goal

Build a delivery pipeline that turns a version-controlled change into a tested, traceable, and reversible deployment.

## What I want to practise

- repository and branching conventions
- automated linting and tests
- build artifacts and container images
- self-hosted versus managed runners
- secret handling
- environment promotion and approvals
- deployment health checks
- rollback after a failed release

## Intended outcome

A small application should move from commit to a lab environment without manual file copying. The pipeline should make failure visible, preserve useful logs, and provide a documented recovery path.

## Questions to answer

- Which work belongs in CI and which belongs in deployment automation?
- How should credentials reach a runner without entering the repository?
- What evidence proves a deployment is healthy?
- How quickly can the previous working version be restored?
