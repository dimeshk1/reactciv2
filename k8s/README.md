# React Hello World on EKS - Deployment Runbook

This document captures the deployment flow, validation steps, and troubleshooting actions used for this project.

## 2-Minute Quick Start

Run from this folder:

	cd /k/projects/react-hello-world/k8s

1. Validate manifests:

	kubectl apply --dry-run=server -f . -R

2. Deploy all manifests:

	kubectl apply -f . -R

3. Verify pods and service:

	kubectl get pods -n github-actions
	kubectl get svc -n github-actions

4. Wait for external hostname and open it in browser:

	kubectl get svc react-hello-world -n github-actions -w

5. Quick health checks:

	kubectl get endpointslice -n github-actions
	kubectl logs -n github-actions -l app=react-hello-world --tail=100

If EXTERNAL-IP stays pending, check subnet tags in AWS (see "Issues We Hit and Fixes" below).

## Architecture Summary

- App is a React static build served by Nginx.
- Docker image is pushed to Amazon ECR.
- Kubernetes manifests deploy to EKS namespace: github-actions.
- Service type is LoadBalancer, exposed through AWS NLB (internet-facing).

## Files in This Folder

- namespace.yaml
- serviceaccount.yaml
- role.yaml
- rolebinding.yaml
- deployment.yaml
- service.yaml

## Key Config Decisions

- Namespace for workloads: github-actions
- Container port: 80 (Nginx)
- Service port: 80
- Service targetPort: 80
- Service annotation for public LB:
  service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing

## Deploy Order

Run from this directory:

	cd /k/projects/react-hello-world/k8s

Apply all resources:

	kubectl apply -f . -R

## Validation Checklist

1. Validate manifests before apply (client + server):

	kubectl apply --dry-run=client --validate=true -f . -R
	kubectl apply --dry-run=server -f . -R

2. Confirm namespace:

	kubectl get ns

3. Confirm workload health:

	kubectl get pods -n github-actions
	kubectl describe pod -n github-actions <pod-name>

4. Check service and endpoints:

	kubectl get svc -n github-actions
	kubectl get endpointslice -n github-actions
	kubectl describe svc react-hello-world -n github-actions

5. Read logs:

	kubectl logs -f -n github-actions <pod-name>
	kubectl logs --previous -n github-actions <pod-name>

## Issues We Hit and Fixes

### 1) Invalid labels with spaces

Symptoms:
- apply failed on Role and RoleBinding with metadata.labels validation errors.

Cause:
- Label value used human-readable text with spaces.

Fix:
- Move description from labels to annotations.

### 2) Resources deployed to wrong namespace

Symptoms:
- Pods showed up in default instead of github-actions.

Cause:
- Several manifests were set to namespace: default.

Fix:
- Updated Deployment, Service, Role, RoleBinding to namespace: github-actions.

### 3) Pods running but not ready / restarts

Symptoms:
- READY 0/1 with probe failures.

Cause:
- Probes and Service targeted 5173, but Nginx image listens on 80.

Fix:
- Changed containerPort/readiness/liveness to 80.
- Changed Service targetPort to 80.

### 4) EXTERNAL-IP stayed pending

Symptoms:
- LoadBalancer service stuck in pending.
- Events showed subnet tag lookup failures.

Cause:
- EKS VPC subnets were missing required Kubernetes tags.

Fix:
- Kept service as internet-facing.
- Tagged EKS subnets with:

	kubernetes.io/role/elb=1
	kubernetes.io/cluster/react-hello-world-dev=shared

After tagging, AWS provisioned an internet-facing NLB and service got external hostname.

## AWS Checks Used

Get cluster VPC and subnet IDs:

	aws eks describe-cluster --name react-hello-world-dev --region us-east-1 \
	  --query "cluster.resourcesVpcConfig.{vpcId:vpcId,subnetIds:subnetIds}" --output table

Inspect subnet tags/public mapping:

	AWS_PAGER="" aws ec2 describe-subnets --region us-east-1 --subnet-ids <subnet-ids...>

Tag subnets for public LB discovery:

	aws ec2 create-tags --region us-east-1 --resources <subnet-ids...> \
	  --tags Key=kubernetes.io/role/elb,Value=1 Key=kubernetes.io/cluster/react-hello-world-dev,Value=shared

## Accessing the App

Get external LB DNS:

	kubectl get svc react-hello-world -n github-actions

Then open the EXTERNAL-IP hostname in browser using HTTP.

If browser shows DNS_PROBE_FINISHED_NXDOMAIN but CLI works:
- Flush local DNS cache.
- Clear browser DNS cache.
- Retry from another network or resolver.

## Cleanup Commands

If old resources remain in default namespace:

	kubectl delete deployment react-hello-world -n default
	kubectl delete service react-hello-world -n default

## CI/CD Notes

- CI builds image and pushes a unique SHA tag to ECR.
- CD resolves the pushed SHA tag to its digest and rolls out EKS using image@sha256.
- Runtime deployment is immutable (digest-based) with no dependency on mutable tags.

