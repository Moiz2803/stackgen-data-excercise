# Context layer — engineering handoff

Human-readable answers to Questions A and B are in `ANSWERS.md`. Detailed machine-readable results and supporting evidence are generated under `output/`.

## Run

Requires Python 3.10+ and only the standard library.

```bash
python3 tools/check_inputs.py
python3 src/run.py
python3 -m unittest discover -s tests -v
```

Generated files are written to `output/question_a.json`, `output/question_b.json`, and `output/issues.json`. Optional query overrides are available with `--tenant`, `--environment`, and `--application`.

## Model and design

I used a small in-memory evidence graph because the exercise is mostly about identity and relationship semantics rather than storage. Nodes represent cloud resources, Terraform records, Kubernetes objects, and catalog applications. Edges represent explicit or narrowly derived relationships such as `managed_binding`, `data_reference`, `controlled_by`, `scheduled_on`, `backed_by`, `declares_workload`, and `declares_dependency`. Every important entity/edge retains a source record reference and observation time through its evidence object.

Identity is intentionally source-aware and scoped. AWS resources are keyed by tenant + account + region + resource type + provider ID. Kubernetes objects are keyed by tenant + cluster + kind + UID. Catalog applications use tenant + service ID. Terraform records use tenant + workspace + resource address; they are *related* to a cloud resource by scoped provider ID rather than treated as the same record. This prevents plausible but unsafe joins on names or private IPs. It also prevents the deliberately repeated EC2 ID in another tenant, the repeated private IP across environments, and reused Kubernetes names from contaminating ACME production results.

Two rules are central. First, **identity requires an authoritative identifier inside its declared scope**: names, tags, and IP addresses are attributes, not cross-source keys. Second, **absence only has meaning inside a successful, complete declared scope**: an unavailable collection is never interpreted as empty, and a complete Terraform workspace does not imply coverage of other workspaces.

## Consequential decisions

The first decision was to treat Terraform `mode=managed` as management evidence and `mode=data` only as a reference. The alternative would be to call any state appearance “managed,” which would incorrectly classify instance `...103`. I would revisit this if a future source supplied Terraform configuration/import metadata with different lifecycle semantics.

The second decision was to resolve a Kubernetes Node to EC2 only from `providerID`, combined with the trusted cluster account/region scope. I deliberately did not fall back to node name or InternalIP. The fixture contains a Node without `providerID` and two production EC2 instances sharing `10.0.9.9`; an IP fallback could create an arbitrary binding. I would add another resolver only if it came from an authoritative mapping source with explicit scope and uniqueness guarantees.

## Verification

I checked whether the old production Terraform snapshot should still be treated as current. Using the manifest's fixed evaluation time, it is well outside its 86,400-second freshness budget. I retained its bindings because they are still supplied evidence, but the query marks them stale and states that they do not establish current management. Tests cover a real managed match, the `mode=data` counterexample, the shared-account tenant collision, the unresolved Node, and rebuild stability. I also checked determinism properly rather than assuming it: relations are held in a set, and comparing two builds inside one process hid the fact that set iteration order varies with `PYTHONHASHSEED`, so `question_b.json` churned between runs. Every relation read now goes through a sorted accessor, and the test spawns two subprocesses with different seeds and diffs the emitted files. The same pass surfaced a managed state record for `i-...099` whose provider object never appears in a complete inventory; the model was asserting that resource existed on state evidence alone, so it is now reported as an unresolved link.

## Readiness and next steps

A read-only internal consumer may rely on the output to trace *supplied evidence*: which running production instances were observed by AWS, whether the supplied Terraform workspace contained a managed binding at its observation time, the catalog-declared application owner/dependency, and Kubernetes workload-to-node-to-EC2 paths where authoritative links exist. Consumers should not interpret these answers as live state, traffic flow, exhaustive ownership, or proof that an instance is unmanaged globally.

The highest-risk gap is coverage across independently collected sources: the production Terraform evidence is stale and only one workspace is supplied. The next changes I would prioritize are (1) a resolver registry with explicit per-source identity contracts and richer disagreement handling, and (2) persisted normalized tables/SQLite plus schema validation so larger extracts can be queried and audited without changing semantics. I would also expose collection health directly in a query API before supporting operational automation.

Time use was roughly: 35 minutes reading/validating inputs, 80 minutes modeling and ingestion, 55 minutes queries/issues, 45 minutes tests and edge cases, and 25 minutes cleanup/documentation. I intentionally stopped at a deterministic local rebuild: no incremental history, database, UI, live connectors, or generalized ontology.
