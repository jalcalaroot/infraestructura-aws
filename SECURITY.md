# Security Policy

This is a personal learning/reference infrastructure project (CloudFormation + AWS, deployed via GitHub Actions), not intended for production use. There are no supported version branches — only `main` is maintained.

## Reporting a Vulnerability

If you find a security issue — an exposed secret, a misconfigured resource, an overly permissive IAM policy, or a problem in a deploy workflow — please report it privately using [GitHub's private vulnerability reporting](../../security/advisories/new) instead of opening a public issue.

## Scope

- This repository's CloudFormation templates, the AWS resources they define, and the GitHub Actions deploy workflows
- Accidentally committed credentials or parameter files with real values

Out of scope: vulnerabilities in AWS services themselves — please report those to AWS.
