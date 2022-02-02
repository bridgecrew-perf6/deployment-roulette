Some pods stay in Pending status after deploying bloatware.yml. To see what's the problem we describe one pending pod and get the following

```kubectl describe pod bloaty-mcbloatface-58c98b98d8-qxfcr
Name:           bloaty-mcbloatface-58c98b98d8-qxfcr
Namespace:      udacity
Priority:       0
Node:           <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-ln4h7 (ro)
Conditions:
  Type           Status
  PodScheduled   False
Volumes:
  kube-api-access-ln4h7:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  33s (x3 over 2m3s)  default-scheduler  0/2 nodes are available: 2 Insufficient cpu.
  ```

Basically, it is telling us that there are not enough nodes in the cluster. The solution is to increase the maximum number of nodes and set up the autoscaling in the cluster.

To increase the maximum number of nodes, we modify the terraform code:
```
module "project_eks" {
  source             = "./modules/eks"
  name               = local.name
  account            = data.aws_caller_identity.current.account_id
  private_subnet_ids = module.vpc.private_subnet_ids
  vpc_id             = module.vpc.vpc_id
  nodes_desired_size = 2
  nodes_max_size     = 4
  nodes_min_size     = 1

  depends_on = [
    module.vpc,
  ]
}
```
And to set up the autoscaling, we do the following
```
eksctl utils associate-iam-oidc-provider --cluster udacity-cluster --approve --region=us-east-2
```
```
eksctl create iamserviceaccount --name cluster-autoscaler --namespace kube-system --cluster udacity-cluster --attach-policy-arn "arn:aws:iam::<account-id>:policy/udacity-k8s-autoscale" --approve --region us-east-2
```
```
kubectl apply -f cluster-autoscale.yml
```