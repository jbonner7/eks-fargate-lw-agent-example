#Sample EKS Fargate Deployment. This attempts to deploy the Lacework Agent as it's own container in a pod rather than a sidecar

#Spin up EC2 for master node (optional)

#Download AWS CLI
```curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"```

"which aws"

```./aws/install --bin-dir --install-dir /usr/bin/aws-cli --update```
OR
```./aws/install --bin-dir /usr/bin --install-dir /usr/bin/aws-cli --update```


#Install kubectl

```curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"```

```chmod +x ./kubectl```

#Add kubectl to $PATH

```mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin```

#Check version

```kubectl version --short --client```

#Install eksctl

```curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp```

```sudo mv /tmp/eksctl /usr/bin```


#Deploy EKS Cluster

```eksctl create cluster -f eks_fargate_cluster_deploy.yaml```

#For managing the cluster within the portal, you either need to create the cluster with the same IAM permissions used to login to the portal, or you must add your portal creds to the configMap
#Download the sample full access yaml that will create a clusterrole, group, and binding for the portal user

```wget https://s3.us-west-2.amazonaws.com/amazon-eks/docs/eks-console-full-access.yaml```

```kubectl apply -f eks-console-full-access.yaml```

#ConfigMap <add sso user/role>
https://aws.amazon.com/premiumsupport/knowledge-center/eks-kubernetes-object-access-error/

```kubectl edit configmap aws-auth -n kube-system```


#Multiple Roles - Copy this format and paste into auth ConfigMap. Replace the two IAM roles that were automatically created and added to the auth ConfigMap. Add the portal user at the bottom

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::<Account_Arn>:role/<auto_created_role>
    username: system:node:{{EC2PrivateDNSName}}
    groups:
    - system:bootstrappers
    - system:nodes
    - eks-console-dashboard-full-access-group
  - rolearn: arn:aws:iam::<Account_Arn>:role/<auto_created_role>
    username: system:node:{{SessionName}}
    groups:
    - system:bootstrappers
    - system:nodes
    - system:node-proxier
  - rolearn: arn:aws:iam::<Account_Arn>:role/portal role
    username: <user>
    groups:
    - system:masters
    - eks-console-dashboard-full-access-group
 ```

If user, not role, add the following and replace the last section with:

 ```yaml 
mapUsers: |
  - userarn: arn:aws:iam::XXXXXXXXXXXX:user/testuser
    username: testuser
    groups:
    - system:bootstrappers
    - system:nodes
    - eks-console-dashboard-full-access-group
```

#Create Fargate profile and namespace for deploying pods. In the sample YAML files, the namespace is "fargate-ns"

```eksctl create fargateprofile  --cluster jb-fargate-cluster  --name test  --namespace fargate-ns```

#Deploy lw-agent ConfigMap

```kubectl apply -f clusterrole_for_agent.yaml```

#Deploy Fargate pod for first app. This will be the app that runs the agent as a sidecar. In the YAML file, this is "app"

```kubectl apply -f eks_fargate_pod_spec.yaml```

#Deploy Fargate pod for second app. This will be the app that runs the agent in it's own container and deploys a second nginx container in the pod. In the YAML file, this is "app2"

```kubectl apply -f agent_container_pod_spec.yaml```

#Deploy an ALB in AWS to reach your nginx app

```kubectl expose pod <app> --port=80 --type=LoadBalancer -n fargate-ns```
