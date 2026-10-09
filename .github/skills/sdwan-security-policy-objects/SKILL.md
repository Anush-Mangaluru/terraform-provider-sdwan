---
name: sdwan-security-policy-objects
description: 'Use when planning or implementing nac-sdwan issue #1301 in terraform-provider-sdwan: 20.18 Security Object Group and Rule Set policy-object parcels, NGFW policy references, schemas, generated provider code, tests, and validation.'
argument-hint: 'Optionally specify plan or implement, and provide the 20.18 API capture location'
---

# SD-WAN Security Policy Objects

## Purpose and Scope

Guide the provider implementation for nac-sdwan issue #1301 in the `.2018_OBJGRP_RULESET` checkout. The provider scope is:

- Security Object Group and Security Rule Set resources and data sources under the `policy-object` feature profile.
- NGFW policy references for object groups and rule sets, including the `isRuleSet` behavior.
- Generated docs, examples, provider registrations, and focused provider tests.

Do not expand this work into the Terraform module, data model, Robot tests, or `nac-tool` import unless the user asks. Do not edit files until the user explicitly says to proceed or implement. A request to plan or save this skill is not implementation authorization.

## Known Starting Point

Recheck the checkout and working tree before implementation; the expected provider checkout is `terraform-provider-sdwan` on branch `ruleset`.

The NGFW definition is `gen/definitions/profile_parcels/embedded_security_ngfw.yaml`; it exposes `rule_set_list_ids` and keeps the legacy `sourceSecurityGroup` / `destinationSecurityGroup` references version-gated. The schema models for the new parcels are `gen/models/profile_parcels/policy_object_security_object_group.json` and `gen/models/profile_parcels/policy_object_security_rule_set.json`, captured from the designated 20.18 Manager.

The live Manager confirmed these POST schema and parcel paths:

- Security Object Group slug `security-object-group`; schema `/dataservice/v1/feature-profile/sdwan/policy-object/security-object-group/schema?schemaType=post`; parcel endpoint `/v1/feature-profile/sdwan/policy-object/%v/security-object-group`.
- Security Rule Set slug `security-rule-set`; schema `/dataservice/v1/feature-profile/sdwan/policy-object/security-rule-set/schema?schemaType=post`; parcel endpoint `/v1/feature-profile/sdwan/policy-object/%v/security-rule-set`.

The Object Group schema requires `data.sequenceIpType` and `data.entries`; entries support IPv4/IPv6 prefixes, FQDNs, geo-locations, ports, and references to corresponding lists. The Rule Set schema requires `data.sequences`; each sequence requires `sequenceName`, `sequenceIpType`, `sequenceId`, and `action`, and supports source/destination object-group references among its match fields. Continue using the captured models as the authoritative schema; do not replace them with guessed definitions. The sibling module still has a TODO in `sdwan_features_embedded_security.tf`; it is out of scope here.

Issue #1259 is the companion NGFW/Embedded Security schema-bump track and should be checked before changing NGFW schema support. It documents the NGFW schema endpoint as `/dataservice/v1/feature-profile/sdwan/embedded-security/unified/ngfirewall/schema?schemaType=post` and the Embedded Security policy endpoint as `/dataservice/v1/feature-profile/sdwan/embedded-security/policy/schema?schemaType=post`. The issue's implementation notes indicate that the NGFW 20.18 schema adds `sequences[].isRuleSet` and `entries.sourceObjectGroup` / `entries.destinationObjectGroup`; the legacy security-group fields are absent from 20.18 but should be retained for compatibility, sent only to Managers below 20.18. The relationship between legacy security groups and object groups is inferred from matching `refId` shapes, not confirmed by the schema, so do not describe them as equivalent without confirmation.

`entries.ruleSetList` exists in both the 20.15 and 20.18 NGFW schemas but is not currently exposed by the definition. Expose it with the Rule Set parcel references and wire `isRuleSet` according to the verified 20.18 payload contract. Gate newly introduced attributes with the repository's MinVersion 20.18 convention. Keep 20.15 compatibility checks in scope for this NGFW work; the 20.18-only object resources/data sources themselves use the `SDWAN_2018` test tag.

## Parallel Work and Conflict Avoidance

As of the 2026-10-08 review, CiscoDevNet PR #778 (`https://github.com/CiscoDevNet/terraform-provider-sdwan/pull/778`) is open against `main` and implements much of issue #1259. Its diff overlaps the NGFW model and definition, generated resource/data-source code, tests, examples, and docs: 20.18 sequence fields, IPv6 match support, source/destination object-group references, legacy security-group version gating/deprecation, and a generator fix. Do not duplicate or independently rework those NGFW changes. Before implementation, recheck the PR state and compare its merged result with this checkout; prefer rebasing/porting onto the merged change before touching overlapping NGFW files.

PR #778 does not add the new `policy-object` Object Group or Rule Set parcel resources/data sources and does not expose `entries.ruleSetList` as `rule_set_list_ids`. Those remain the distinct issue #1301 work. If the PR is still open, keep any first-pass parcel work isolated from the NGFW definition/model/generated files, then add `rule_set_list_ids` and related `is_rule_set` integration after coordinating with the PR owner or once the PR is merged.

## Lab Access Before Retesting

The API discovery is complete for the current target Manager. To repeat or extend live testing, use the following without requesting credentials in chat:

1. Use the user's designated 20.18 test Manager at `https://10.50.202.35`, or sanitized API captures for both objects. Credentials must remain in the user's local environment. The matching `lab.env` block is currently commented out, while another Manager is selected by the active exports; explicitly select the intended 20.18 settings before testing and verify the Manager version first.
2. The designated Manager is `https://10.50.202.35`; its corresponding block in `lab.env` is commented out, while other exports select a different Manager. Select the correct local settings and verify the version before acceptance tests.
3. The provider's local `.env` or environment variables should supply `SDWAN_URL`, `SDWAN_USERNAME`, and `SDWAN_PASSWORD`; acceptance tests need `SDWAN_2018` and `TF_ACC` set. Never print, copy into the skill, or request these secrets in chat.
4. The API create/read flows for both parcels were validated by focused acceptance tests against this Manager. If future schema changes are required, capture the updated schema and valid payload from the Manager before editing model JSON.

## Workflow

1. Inspect the provider checkout's git status and local instructions. Preserve all pre-existing user changes. Recheck PR #778's state; this `.2018_OBJGRP_RULESET` implementation includes the fetched patch as a prerequisite, so confirm its commit/merge status to avoid duplicating #1259 changes against upstream.
2. Use the captured model JSON and confirmed endpoint paths above. For future API changes, verify `refId` representation and how `isRuleSet` distinguishes a rule-set match from other match entries before editing provider code.
3. Add each confirmed model JSON under `gen/models/profile_parcels/` and its generator definition under `gen/definitions/profile_parcels/`. Follow neighboring policy-object parcel conventions. Set `minimum_version: 20.18.0`, `test_tags: [SDWAN_2018]`, the verified REST endpoint, ID field, examples, and prerequisites. Do not derive fields or endpoint slugs by analogy alone.
4. Update the embedded-security NGFW model and definition for the verified source/destination object-group references and rule-set list. Keep the old `source_security_group_list_ids` and `destination_security_group_list_ids` attributes as deprecated compatibility inputs; follow #1259's version-gating pattern so they are sent only to Managers below 20.18. Add separate object-group attributes and gate the new NGFW fields at MinVersion 20.18. Ensure `isRuleSet` is represented and serialized as required by the API. Test the compatibility path against 20.15 as well as the new behavior against 20.18.
5. Run the repository generator for each parcel using `make gen NAME="<definition name>"`. Review generated provider registration, resource/data-source implementations, docs, examples, and tests. Avoid hand-editing generated sections unless the repository generator requires a source-template change.
6. Add or adjust tests for parcel create/read/data-source lookup and NGFW references, with valid 20.18 test values. Keep 20.18 acceptance tests gated by `SDWAN_2018` and do not make tests depend on invented or arbitrary UUIDs when resources can provide their IDs.
7. Validate with `go build ./...`, `go vet ./internal/provider/`, and focused test compilation (for example, `go test ./internal/provider/ -run '^$'`). Run focused acceptance tests against the 20.18 Manager when its local test configuration is available. Review `git diff` and confirm generation did not introduce unrelated changes.

## Completion Criteria

- Both resources and data sources use verified endpoints and schemas and pass provider validation.
- NGFW references serialize the expected object-group and rule-set structures, including `isRuleSet`.
- Existing supported 20.15 behavior is not unintentionally regressed.
- Generated documentation/examples and focused tests are present and current.
- Build, vet, and focused compile checks pass; acceptance-test results or environmental blockers are reported clearly.
- The downstream module/data-model/Robot/import work is explicitly left out unless separately requested.

## Latest Validation

On the designated 20.18 Manager, focused acceptance tests passed for Object Group create/read, Rule Set create/read, and NGFW Rule Set reference write/readback (`rule_set_list_ids` with `is_rule_set = true`). `go build ./...`, `go vet ./internal/provider/`, provider test compilation, and CRLF-aware `git diff --check` also passed. Repeat these checks after later changes.
