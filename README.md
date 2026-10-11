# k8s-infra

Shared Kubernetes resources for the kubeadm cluster.

| Repository | Owns |
| --- | --- |
| [terraform](https://github.com/hgfranco/terraform) | AWS networking, IAM, ECR, ACM certificates and DNS |
| [ansible](https://github.com/hgfranco/ansible) | Node provisioning and cluster controllers |
| k8s-infra | Shared ALB routing and cluster-wide storage configuration |
| Application repositories | Images, Deployments, Services, app configuration, policies and PVCs |

## Resources

- `clusters/kubernetes/ingress.yaml`: the existing ALB Ingress routing Spotify, henryfranco.dev and henryfranco.com.
- `clusters/kubernetes/storageclass.yaml`: the existing encrypted gp3 StorageClass, with Retain and WaitForFirstConsumer.

The Ingress keeps its existing name and namespace: `spotify-now-playing/spotify-now-playing`. Both application Services are in that namespace because Ingress backends refer to Services in the same namespace. The names preserve the existing ALB; repository ownership does not require a new load balancer.

## Moving source ownership

Merge this repository's infrastructure PR before the Spotify cleanup PR. Removing the old YAML files from Spotify is a source move only. **Do not delete the live Ingress, StorageClass, PVC or EBS volume. Do not prune resources as part of this move.** The current setup applies manifests manually and does not automatically reconcile or delete resources when files change.

Merging a PR does not deploy anything.

## Provisioning order

1. Provision the AWS infrastructure with Terraform and the cluster/controllers with Ansible.
2. Ensure the `spotify-now-playing` namespace exists, then apply the StorageClass before the app's PVC:
   `kubectl apply -f clusters/kubernetes/storageclass.yaml`.
   The StorageClass itself does not create an EBS volume.
3. Publish application images and apply the app manifests from their respective repositories.
4. Before applying this Ingress, verify both Services have ready endpoints and all three ACM certificates are ISSUED in us-east-1. The personal site certificate and workload are prerequisites for the henryfranco.dev route.
5. Review and apply the shared Ingress:

```bash
kubectl apply --dry-run=server -f clusters/kubernetes/ingress.yaml
kubectl diff -f clusters/kubernetes/ingress.yaml
kubectl apply -f clusters/kubernetes/ingress.yaml
kubectl get ingress spotify-now-playing -n spotify-now-playing
```

A diff exit code of 1 means differences were found. Review them before applying. Ingress hostname-based discovery selects ACM certificates; Terraform owns those certificates.

6. Test HTTPS against the ALB before publishing henryfranco.dev in DNS:

```bash
ALB_HOST=$(kubectl get ingress spotify-now-playing -n spotify-now-playing -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -fsS --connect-to "henryfranco.dev:443:${ALB_HOST}:443" https://henryfranco.dev/
```

7. Enable the Terraform personal-site DNS switch only after the test succeeds. Preserve old DNS values for rollback.

## Changes and validation

All changes go through a PR. GitHub Actions validates the Kubernetes schemas. Schema validation does not verify live Services, ACM certificate readiness or ALB health; those are checked during deployment. Only the repository owner merges, using the owner-only PR bypass when GitHub presents that option. Direct pushes to the default branch remain blocked.

## henryfranco.com

Both personal-site domains route to the same henryfranco-dev Service. Visitors
keep the domain they entered. Terraform owns the separate .com ACM certificate
and staged Cloudflare DNS cutover.

1. Apply [Terraform PR #23](https://github.com/hgfranco/terraform/pull/23) in the kubernetes workspace and wait for the .com certificate to be ISSUED in us-east-1.
2. Apply this Ingress using the review commands above.
3. Test .com directly against the ALB before enabling its Terraform DNS switch:

```bash
curl -fsS --connect-to "henryfranco.com:443:${ALB_HOST}:443" https://henryfranco.com/healthz
curl -fsS --connect-to "henryfranco.com:443:${ALB_HOST}:443" https://henryfranco.com/
curl -fsS --connect-to "henryfranco.com:443:${ALB_HOST}:443" https://henryfranco.com/about/
```

4. Follow [the Terraform cutover guide](https://github.com/hgfranco/terraform/blob/master/environments/public_endpoint/henryfranco-com.md) to import/update the apex DNS record. Verify both public domains afterward.

There is no redirect between .com and .dev. The existing HTTP-to-HTTPS rule
continues to preserve the hostname. No www hostname is added.

## Retire the old Spotify subdomain

This Ingress removes its old host rule and TLS host. Wait for the ALB controller
to detach that certificate before applying [Terraform #24](https://github.com/hgfranco/terraform/pull/24), which removes the unused DNS CNAME and dedicated certificate. The active Spotify apex route stays in place.
