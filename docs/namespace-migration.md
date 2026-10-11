# Separate application namespaces without an abrupt cutover

This migration uses an additional ALB temporarily. The legacy Ingress keeps
serving every existing route while the new ALB and namespace are prepared.
Expect temporary charges for a second ALB and two additional website Pods.

## 1. Prepare the namespace and image

Merge the companion PRs in k8s-infra, henryfranco.dev and terraform.
Check the new website's 0.5.0 image exists before applying its workload.
If the stylesheet fix is released as a newer image, update the Deployment
image through a PR before starting this migration; do not downgrade a live
website to an older release.

Save the current routing and workload for rollback:

```bash
kubectl get ingress spotify-now-playing -n spotify-now-playing -o yaml > /tmp/legacy-app-ingress.yaml
kubectl get deployment henryfranco-dev -n spotify-now-playing -o yaml > /tmp/legacy-website-deployment.yaml
kubectl get service henryfranco-dev -n spotify-now-playing -o yaml > /tmp/legacy-website-service.yaml
kubectl get ingress spotify-now-playing -n spotify-now-playing -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}'
kubectl apply --dry-run=server -f clusters/kubernetes/henryfranco-namespace.yaml
kubectl apply -f clusters/kubernetes/henryfranco-namespace.yaml
```

Apply the website manifests from henryfranco.dev using its namespace migration
guide. Wait for both Pods to be ready in henryfranco. Keep the old Pods running.
The namespace default-deny policy is additive with the app's VPC ingress rule.

## 2. Create new grouped routing

Do not rename, annotate with group.name, or delete the legacy Ingress.
The active manifests use new Ingress names, so these commands create additional
routing rather than mutating the live legacy route:

```bash
kubectl apply --dry-run=server -f clusters/kubernetes/ingress.yaml
kubectl apply --dry-run=server -f clusters/kubernetes/henryfranco-ingress.yaml
kubectl diff -f clusters/kubernetes/ingress.yaml
kubectl diff -f clusters/kubernetes/henryfranco-ingress.yaml
kubectl apply -f clusters/kubernetes/ingress.yaml
kubectl apply -f clusters/kubernetes/henryfranco-ingress.yaml
kubectl get ingress -n spotify-now-playing
kubectl get ingress -n henryfranco
```

Diff exit code 1 means changes were found. Review before applying.
Both new Ingresses must report the same new ALB hostname. The controller creates
kubernetes-public-apps and discovers all three certificates from TLS hosts.
Service-level health checks override the Ingress default: /healthz for the
website and /api/status for Spotify.

```bash
ALB_HOST=$(kubectl get ingress henryfranco -n henryfranco -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
SPOTIFY_ALB_HOST=$(kubectl get ingress spotify-public -n spotify-now-playing -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
test -n "$ALB_HOST" && test "$ALB_HOST" = "$SPOTIFY_ALB_HOST"
aws elbv2 describe-load-balancers --region us-east-1 --names kubernetes-public-apps --query 'LoadBalancers[].{ARN:LoadBalancerArn,DNS:DNSName,State:State.Code}'
kubectl get targetgroupbindings -n henryfranco
kubectl get targetgroupbindings -n spotify-now-playing
```

Use each new TargetGroupBinding's ARN with aws elbv2 describe-target-health.
Require healthy targets for both Services. Local traffic policy means a worker
without a local ready website Pod may be unhealthy; at least one must be healthy
and the two website replicas should be spread across workers.

## 3. Test directly, before changing DNS

```bash
curl -fsS --max-time 20 --connect-to "whatishenrylisteningto.com:443:${ALB_HOST}:443" https://whatishenrylisteningto.com/api/status
for domain in henryfranco.com henryfranco.dev; do
  for path in / /about/ /resume/ /healthz; do
    curl -fsS --max-time 20 --connect-to "${domain}:443:${ALB_HOST}:443" -o /dev/null "https://${domain}${path}"
  done
done
```

Verify HTTPS succeeds without disabling certificate verification and no personal
domain redirects to another domain. Check the actual website content and image
version, not only status codes. Repeat requests to detect intermittent failures.

## 4. Cut over DNS with Terraform

Follow environments/public_endpoint/shared-alb-migration.md in terraform.
Keep shared_alb_dns_enabled false until the checks above succeed. The enabled
switch changes all three existing DNS records in place to the new ALB.
Review/apply a full plan without -target. Cloudflare stays Full (strict).

Observe public traffic and verify all domains. Keep the legacy ALB and old
website running for resolver caches and rollback, at least 24 hours after stable
cutover. Confirm no other DNS consumer still points to the legacy ALB.

## 5. Retire legacy resources

Only after the observation window and explicit verification:

```bash
kubectl delete ingress spotify-now-playing -n spotify-now-playing
kubectl delete deployment henryfranco-dev -n spotify-now-playing
kubectl delete service henryfranco-dev -n spotify-now-playing
kubectl delete poddisruptionbudget henryfranco-dev -n spotify-now-playing
kubectl delete networkpolicy henryfranco-dev-ingress -n spotify-now-playing
```

Deleting the legacy Ingress removes its ALB through the controller.
Do not delete spotify-public, the Spotify namespace, Spotify Deployment,
StorageClass, PVC, or EBS volume. Watch controller logs and verify all three
public domains again after cleanup. The two active Ingresses remain in the
public-applications group, sharing one ALB.

## Rollback

Before cleanup: return Terraform's DNS switch to false, review/apply the three
reverse DNS updates, and keep both old resources serving until caches settle.

After cleanup: restore the legacy Ingress snapshot from
migrations/personal-site-namespace/legacy-ingress.yaml and restore the old website
workload from the previous app manifests or saved resource specs. Remove generated
status, resourceVersion, uid and other server metadata before applying saved YAML.
Wait for ready targets and verify all routes directly. The recreated legacy ALB
may have a different hostname; update Terraform's legacy ALB inputs before
switching DNS back. Do not simply flip the switch toward a deleted ALB.

Controller IngressGroup documentation:
https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/#ingressgroup
