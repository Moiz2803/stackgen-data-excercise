# Assessment Answers

This document provides the human-readable answers to Questions A and B. The corresponding machine-readable results, including query scope, evaluation time, supporting record references, freshness, and limitations, are in `output/question_a.json`, `output/question_b.json`, and `output/issues.json`.

## Question A — Terraform management evidence

For tenant `acme` in the `prod` environment, the AWS inventory observes five running EC2 instances at the evaluation time.

| EC2 instance | Result | Terraform evidence |
| --- | --- | --- |
| `i-00000000000000101` | Managed binding found | `module.workers.aws_instance.pool[0]` in workspace `acme-prod-core` |
| `i-00000000000000102` | Managed binding found | `module.workers.aws_instance.pool[1]` in workspace `acme-prod-core` |
| `i-00000000000000103` | No managed binding found in supplied scope | Present only as `data.aws_instance.lookup` (`mode=data`) |
| `i-00000000000000104` | No managed binding found in supplied scope | No qualifying managed-resource binding in the supplied workspace |
| `i-00000000000000105` | No managed binding found in supplied scope | No qualifying managed-resource binding in the supplied workspace |

Instances `...101` and `...102` therefore have managed-resource bindings in the supplied Terraform state. Instance `...103` is present only as a Terraform data source. I treat `mode=data` as reference evidence rather than lifecycle-management evidence, so it is not classified as a managed binding. No managed binding was found for `...104` or `...105` within the supplied relevant Terraform scope.

The production Terraform snapshot was observed at `2026-09-02T09:00:00Z`, while the manifest evaluation time is `2026-09-16T12:00:00Z`. This exceeds the snapshot's 86,400-second freshness budget. The bindings for `...101` and `...102` establish what the supplied Terraform state showed at its observation time, but they do not by themselves prove current Terraform management. Likewise, absence from this supplied workspace does not prove an instance is unmanaged by every possible workspace or management system.

Detailed evidence and source references are in `output/question_a.json`.

## Question B — `payments-api` application context and responsibility

For tenant `acme` in production, the service catalog identifies `team-payments` as the application team for `payments-api`.

The catalog declares the `payments-api` Kubernetes Deployment as the application's workload. Kubernetes `ownerReferences` connect that Deployment to its ReplicaSet and Pods. Each Pod's `spec.nodeName` identifies its Node, and each resolved Node's `providerID`, combined with the trusted cluster scope, identifies the backing EC2 instance.

The supported runtime paths resolve to:

| Workload | Kubernetes Node | Backing EC2 | Infrastructure operator |
| --- | --- | --- | --- |
| `payments-api-7c9d-a` | `ip-10-0-4-118` | `i-00000000000000101` | `team-platform` |
| `payments-api-7c9d-b` | `ip-10-0-4-119` | `i-00000000000000102` | `team-platform` |

One qualification applies to the operator column. For `i-00000000000000101` the AWS inventory tag states `team-platform`, observed `2026-09-16T11:50:00Z`, while the Terraform state records `team-legacy-platform`, observed `2026-09-02T09:00:00Z` and outside its freshness budget. Both statements are retained rather than silently reconciled; the disagreement is reported in `output/issues.json`.

The service catalog separately declares the RDS resource `payments-db` as an application dependency. AWS inventory evidence identifies `team-data-platform` as the infrastructure operator for that database.

The responsibilities are therefore intentionally distinct:

- Application responsibility: `team-payments`
- Runtime EC2 infrastructure operation: `team-platform`
- Declared application dependency: `payments-db`
- Database infrastructure operation: `team-data-platform`

A Pod running on an EC2-backed Node does not make that EC2 instance a declared application dependency. The EC2 relationship describes runtime placement; the RDS relationship is separately declared by the service catalog.

These paths are observation-time evidence rather than proof of live traffic or current readiness. Relationships are resolved only where the supplied catalog references, Kubernetes owner UIDs, `nodeName`, and Node `providerID` support them; names and IP addresses alone are not treated as identity.

Detailed evidence and supporting record references are in `output/question_b.json`. Unresolved links, conflicts, freshness, and source-coverage limitations are reported in `output/issues.json`.
