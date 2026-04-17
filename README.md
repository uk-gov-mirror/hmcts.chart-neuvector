# chart-neuvector

Helm Chart for NeuVector

This chart builds upon the official Neuvector Helm chart adding a job that automates post-install and post-upgrade configuration.
As the chart pulls secrets from Azure keyvault using a secret volume, the changes are not generic enough to be merged to upstream and that is the reason for creating this derived chart.

## Functionalities

After installing Neuvector using the official Neuvector chart, this chart does the following:

1. Accepts the EULA if not accepted already (i.e. because this is an upgrade)
2. Sets the license key if not set already. You can set `config.forceLicenseUpdate` to `true` for force updating license
3. Changes the default admin password if it has not been changed already
4. Sets the Slack webhook URL
5. Sets the admission rules
6. Sets the response rules

## Rule values

The chart now renders admission and network CRDs from values instead of hardcoded templates.

- `rules.admission.defaultRules` contains the chart baseline admission rules.
- `rules.admission.rules` can be used to append environment-specific admission rules.
- `rules.admission.includeDefaultRules` can be set to `false` if an environment needs to replace the baseline.
- `rules.network.defaultRules` contains the chart baseline network CRDs.
- `rules.network.rules` can be used to append environment-specific network CRDs.
- `rules.network.includeDefaultRules` can be set to `false` if an environment needs to replace the baseline.
- `groups` can be used to add environment-specific NeuVector group CRDs alongside network rules.

This split is intentional because Helm replaces arrays during values merges rather than appending them.

For a strict namespace-scoped policy such as `crime-idam`, the current chart baseline is broader than the goal because `rules.network.defaultRules` includes the cluster-wide `allpods` rule and allows more than PostgreSQL and DNS. In that case, disable the baseline network rules in Flux and provide namespace-specific rules instead.

Example Flux values for `crime-idam`:

```yaml
spec:
  values:
    rules:
      network:
        includeDefaultRules: false
        rules:
          - apiVersion: neuvector.com/v1
            kind: NvClusterSecurityRule
            metadata:
              name: crime-idam-egress
              namespace: ""
            spec:
              egress:
                - action: allow
                  applications:
                    - DNS
                  name: crime-idam-dns-egress
                  ports: udp/53,tcp/53
                  priority: 0
                  selector:
                    comment: ""
                    criteria:
                      - key: address
                        op: =
                        value: kube-dns-upstream.kube-system.svc.cluster.local
                    name: KubeDns
                    original_name: ""
                - action: allow
                  applications:
                    - PostgreSQL
                    - SSL
                  name: crime-idam-postgres-egress
                  ports: tcp/5432
                  priority: 0
                  selector:
                    comment: ""
                    criteria:
                      - key: address
                        op: =
                        value: '*.postgres.database.azure.com'
                    name: AzurePostgreSQL
                    original_name: ""
              file: []
              ingress: []
              process: []
              target:
                policymode: Protect
                selector:
                  comment: ""
                  criteria:
                    - key: namespace
                      op: =
                      value: crime-idam
                  name: crime-idam
                  original_name: ""
```

Validate the DNS selector against the cluster's actual DNS service before rollout. In this estate there is evidence of `kube-dns-upstream` under `kube-system`, but some clusters may differ.

For the secrets (e.g. admin password, license key) to be read from Azure keyvault, an Azure managed identity needs to be available.
For more information refer to the documentation related to Pod Identity and Azure provider for CSI driver:

- [Azure Key Vault Provider for Secrets Store CSI Driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure)
- [AAD Pod Identity](https://github.com/Azure/aad-pod-identity)

The automated configuration script uses the Neuvector REST API therefore the controller service should be exposed at least internally
(i.e. without ingress).

## Optional http->https redirect

An optional http->https redirect can be enabled on the manager ingress by setting the following boolean parameter:

```yaml
spec:
  values:
    neuvector:
      manager:
        ingress:
          httpredirect: true
```

`httpredirect` now defaults to `false`, so the chart renders cleanly when the setting is omitted.

## Releases

We use semantic versioning via GitHub releases to handle new releases of this application chart, this is done via automation called Release Drafter. When you merge a PR to master, a new draft release will be created.
More information is available about the [release process and how to create draft releases for testing purposes in more depth](https://hmcts.github.io/ops-runbooks/Testing-Changes/drafting-a-release.html)
