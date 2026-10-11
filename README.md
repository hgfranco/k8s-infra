# k8s-infra

Shared Kubernetes resources for the kubeadm cluster.

| Repository | Owns |
| --- | --- |
| terraform | AWS networking, IAM, ECR, ACM certificates and DNS |
| ansible | Node provisioning and cluster controllers |
| k8s-infra | Application namespaces, shared ALB routing and cluster-wide storage configuration |
| Application repositories | Images, Deployments, Services, app configuration, policies and PVCs |

## Resources

- clusters/kubernetes/henryfranco-namespace.yaml: the personal site's namespace,
  restricted Pod Security settings and default-deny ingress/egress policy.
- clusters/kubernetes/ingress.yaml: spotify-now-playing/spotify-public.
- clusters/kubernetes/henryfranco-ingress.yaml: henryfranco/henryfranco, serving
  henryfranco.dev and henryfranco.com without redirects between them.
- clusters/kubernetes/storageclass.yaml: encrypted gp3 storage configuration.

The two active Ingresses share group.name public-applications and the
controller-owned ALB kubernetes-public-apps. Each Ingress references a Service
in its own namespace. Spotify's workload and EBS storage stay where they are.

## Existing cluster migration

**Follow [the migration guide](docs/namespace-migration.md).** The grouped ALB is
new. The existing spotify-now-playing/spotify-now-playing Ingress remains live
until explicitly deleted after DNS cutover and an observation window. Its
snapshot is in migrations/personal-site-namespace/legacy-ingress.yaml, outside
the active manifests. Do not apply with pruning or delete the legacy Ingress
during preparation. Merging these changes does not deploy anything.

## Fresh cluster

Provision AWS with Terraform and nodes/controllers with Ansible. Create the
Spotify namespace and storage before its PVC, then apply the personal namespace:

```bash
kubectl create namespace spotify-now-playing --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f clusters/kubernetes/storageclass.yaml
kubectl apply -f clusters/kubernetes/henryfranco-namespace.yaml
```

Publish the app images and apply workloads from both application repositories.
Wait for ready Service endpoints and all three ACM certificates in us-east-1.
Then review and apply both Ingresses:

```bash
kubectl apply --dry-run=server -f clusters/kubernetes/ingress.yaml
kubectl apply --dry-run=server -f clusters/kubernetes/henryfranco-ingress.yaml
kubectl apply -f clusters/kubernetes/ingress.yaml
kubectl apply -f clusters/kubernetes/henryfranco-ingress.yaml
```

Verify target health and direct HTTPS before publishing DNS. Use Terraform's
shared_alb_dns_enabled switch only after verification. A fresh cluster does not
need the legacy Ingress or its old website workload.

## Ownership and isolation

Namespace creation is separate from application deployments. The app's policy
permits VPC TCP/8080 for instance-mode ALB traffic; the namespace denies other
traffic by default. Namespace boundaries alone do not isolate networks.
Application Pods do not mount Kubernetes credentials.

IngressGroup shares a routing trust boundary: only trusted cluster operators
should create or modify these Ingresses. Namespace separation is not permission
to give unrelated tenants write access to the shared group.

All changes go through a PR. GitHub Actions validates Kubernetes schemas.
Live endpoint, certificate and target health checks are part of deployment.
Only the repository owner merges. No www hostname is configured.
