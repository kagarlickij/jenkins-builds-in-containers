# Disclaimer
This repo contains code for demo purposes only, it contains hardcoded values and no security hardening, so it never should be used in production

# Demo 1: Kubernetes on EC2 - provisioning with KOPS

## Preparatory steps
1. Create S3 bucket for KOPS state
2. Create Route53 hosted zone
3. Create SSH key
4. Install and configure `aws cli`
5. Install `aws-iam-authenticator`
6. Install `ssm-run`
7. Install `kops`
8. Install `kubectl`
9. Install `kubedecode`
10. Install `helm`

## Create cluster
```
kops create cluster --zones eu-west-1a demo1.demo.kagarlickij.com --state s3://kag-kops-state --ssh-public-key ~/.ssh/kops.pub --yes
```

## Validate cluster
```
kops validate cluster demo1.demo.kagarlickij.com --state s3://kag-kops-state
```

## Check connection
```
kubectl cluster-info
```

# Demo 2: Jenkins inside Kubernetes - deployment with HELM

## Create service account for Tiller
```
kubectl apply --filename=k8s-tiller-service-account.yaml
```

## Install Tiller
```
helm init --service-account tiller
```

## Check Tiller
```
helm version
```

## Install Jenkins
```
helm install stable/jenkins --name jenkins-master --values helm-jenkins-master-values.yaml
```

## Check pod readiness
```
kubectl get pods --watch
```

## Get Jenkins password for admin user
```
printf $(kubectl get secret --namespace default jenkins-master -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

## Open Jenkins URL
```
JENKINS_URL=http://$(kubectl get svc --namespace default jenkins-master --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}"):8080/ && python -m webbrowser $JENKINS_URL
```

## Update Jenkins plugins
Open Jenkins URL and go to Manage Jenkins > Manage Plugins > Updates,

..and select all available updates, than install and restart Jenkins.

## Check Jenkins connection to Kubernetes cluster
Open Jenkins URL and go to Manage Jenkins > Configure system > Cloud > Kubernetes

## Change Kubernetes agent label
Open Jenkins URL and go to Manage Jenkins > Configure system > Cloud > Kubernetes > Kubernetes Pod Template,

..and change label from `jenkins-master-jenkins-slave ` to `k8s-agent`

# Demo 3: Jenkins job execution inside Kubernetes

## Create new pipeline job and insert the following pipeline script
```
pipeline {
    agent {
        node {
            label 'k8s-agent'
        }
    }
    stages {
        stage ('build') {
            steps {
                echo "Hello World!"
            }
        }
    }
}
```

## Start Jenkins job and check agent pods
```
kubectl get pods --watch
```

# Demo 4: EKS - provisioning with CloudFormation

## Create Cluster
```
aws cloudformation create-stack --stack-name demo4p1 --template-body file://aws-eks-cluster.yaml --capabilities CAPABILITY_NAMED_IAM
```

## Wait for provisioning to be completed
```
aws cloudformation wait stack-create-complete --stack-name demo4p1
```

## Check stack status
```
aws cloudformation describe-stacks --stack-name demo4p1 | jq --raw-output '.Stacks | .[] | .StackStatus'
```

## Add cluster to kubeconfig
```
aws eks update-kubeconfig --name demo4
```

## Check connection
```
kubectl cluster-info
```

## Create Nodes
```
aws cloudformation create-stack --stack-name demo4p2 --template-body file://aws-eks-nodes.yaml --capabilities CAPABILITY_NAMED_IAM
```

## Wait for provisioning to be completed
```
aws cloudformation wait stack-create-complete --stack-name demo4p2
```

## Check stack status
```
aws cloudformation describe-stacks --stack-name demo4p2 | jq --raw-output '.Stacks | .[] | .StackStatus'
```

## Apply ConfigMap with IAM roles mappings
```
kubectl apply --filename=k8s-aws-auth-config.yaml
```

## Check nodes
```
kubectl get nodes --watch
```

# Demo 5: Jenkins deployment on EC2 with CloudFormation

## Create instance
```
aws cloudformation create-stack --stack-name demo5 --template-body file://aws-ec2-jenkins.yaml --capabilities CAPABILITY_NAMED_IAM
```

## Wait for provisioning to be completed
```
aws cloudformation wait stack-create-complete --stack-name demo5
```

## Check stack status
```
aws cloudformation describe-stacks --stack-name demo5 | jq --raw-output '.Stacks | .[] | .StackStatus'
```

## Get Jenkins password for admin user
```
ssm-run "cat /var/lib/jenkins/secrets/initialAdminPassword" $(aws cloudformation describe-stacks --stack-name demo5 | jq --raw-output '.Stacks | .[] | .Outputs | .[] | select(.OutputKey == "InstanceId").OutputValue')
```

## Open Jenkins URL
```
JENKINS_URL=http://$(aws cloudformation describe-stacks --stack-name demo5 --region eu-west-1 | jq --raw-output '.Stacks | .[] | .Outputs | .[] | select(.OutputKey == "InstancePublicIp").OutputValue'):8080/ && python -m webbrowser $JENKINS_URL
```

# Demo 6: Jenkins job execution in Kubernetes on EC2 using basic auth

## Switch context to KOPS cluster
```
kubectl config use-context demo1.demo.kagarlickij.com
```

## Install Jenkins Kubernetes plugin
Open Jenkins URL and go to Manage Jenkins > Manage Plugins > Available > Filter > Kubernetes
..and install it with Jenkins restart

## Enable Fixed TCP port for JNLP agents
Open Jenkins URL and go to Manage Jenkins > Configure Global Security > Agents > TCP port for JNLP agents > Fixed > 50000

## Add new 'Username with password' credentials

Username: admin

Password: value from
```
kops get secrets kube -oplaintext --state s3://kag-kops-state
```

Description: kops-basic-auth

## Get list of secrets
```
kubectl get secrets
```

## Decode default token to get ca.crt value
```
kubedecode default-token-$$$$$
```

## Add Kubernetes as a cloud to Jenkins
Open Jenkins URL and go to Manage Jenkins > Configure system > Cloud > Add a new cloud > Kubernetes and enter:

1. Kubernetes URL (`kubectl cluster-info`)

2. Kubernetes server certificate key (from previous step)

3. Check 'Disable https certificate check' option

4. Use `default` for Kubernetes Namespace

5. Click 'Text Connection' button

6. Enter Jenkins URL

7. Enter Jenkins tunnel

## Add Pod Template

1. Name: 'kops-pod'

2. Namespace: 'default'

3. Labels: 'kops-agent'

4. Usage: Use this node as much as possible

## Add Container Template

1. Name: jnlp-slave

2. Docker image: jenkins/jnlp-slave

## Create new pipeline job and insert the following pipeline script
```
pipeline {
    agent {
        node {
            label 'kops-agent'
        }
    }
    stages {
        stage ('build') {
            steps {
                echo "Hello World!"
            }
        }
    }
}
```

## Start Jenkins job and check agent pods
```
kubectl get pods --watch
```

# Demo 7: Jenkins jobs execution in EKS

## Switch context to EKS cluster
```
kubectl config use-context arn:aws:eks:eu-west-1:709237651222:cluster/demo4
```

## Create Jenkins service account
```
kubectl apply --filename=k8s-jenkins-service-account.yaml
```

## Get list of secrets
```
kubectl get secrets
```

## Add new 'Secret text' credentials
Secret: value from
```
kubedecode jenkins-token-$$$$$
```

Description: eks-token

## Add EKS as a Kubernetes cloud to Jenkins
Open Jenkins URL and go to Manage Jenkins > Configure system > Cloud > Add a new cloud > Kubernetes and enter:

1. Kubernetes URL (`kubectl cluster-info`)

2. Kubernetes server certificate key (from previous step)

3. Check 'Disable https certificate check' option

4. Use `default` for Kubernetes Namespace

5. Click 'Text Connection' button

6. Enter Jenkins URL

7. Enter Jenkins tunnel

## Add Pod Template

1. Name: 'eks-pod'

2. Namespace: 'default'

3. Labels: 'eks-agent'

4. Usage: Use this node as much as possible

## Add Container Template

1. Name: jnlp-slave

2. Docker image: jenkins/jnlp-slave

## Create new pipeline job and insert the following pipeline script
```
pipeline {
    agent {
        node {
            label 'eks-agent'
        }
    }
    stages {
        stage ('build') {
            steps {
                echo "Hello World!"
            }
        }
    }
}
```

## Start Jenkins job and check agent pods
```
kubectl get pods --watch
```

# Demo 8: ECS provisioning with CloudFormation

## Create Cluster
```
aws cloudformation create-stack --stack-name demo8 --template-body file://aws-ecs-cluster.yaml
```

## Wait for provisioning to be completed
```
aws cloudformation wait stack-create-complete --stack-name demo8
```

## Check stack status
```
aws cloudformation describe-stacks --stack-name demo8 | jq --raw-output '.Stacks | .[] | .StackStatus'
```

## Check cluster
```
aws ecs describe-clusters --cluster fargate | jq
```

# Demo 9: Jenkins jobs execution in ECS (Fargate)

## Install Jenkins Amazon Elastic Container Service plugin
Open Jenkins URL and go to Manage Jenkins > Manage Plugins > Available > Filter > Amazon Elastic Container Service
..and install it with Jenkins restart

## Add ECS as cloud to Jenkins
Open Jenkins URL and go to Manage Jenkins > Configure system > Cloud > Amazon EC2 Container Service Cloud and enter:

1. Name: fargate

2. Amazon ECS Region Name: eu-west-1

3. ECS Cluster: arn:aws:ecs:eu-west-1:$$$$$$$$$$$$:cluster/fargate

4. Click on Advanced tab

5. Enter Tunnel connection through

6. Enter Alternative Jenkins URL

7. Container Cleanup Timeout: 10

## Add ECS agent template:

1. Label: fargate-agent

2. Template name: fargate

3. Launch type: FARGATE

4. Soft Memory Reservation: 1024

5. CPU units: 512

6. Subnets: `subnet-3c1bbc66, subnet-46bbde20, subnet-860773ce`

7. Security Groups: `sg-e0b51a91`

8. Enable 'Assign Public Ip' option

## Create new pipeline job and insert the following pipeline script
```
pipeline {
    agent {
        node {
            label 'fargate-agent'
        }
    }
    stages {
        stage ('build') {
            steps {
                echo "Hello World!"
            }
        }
    }
}
```

## Start Jenkins job and check fargate tasks
```
aws ecs list-tasks --cluster fargate | jq
```

# Delete resources
```
kops delete cluster --name=demo1.demo.kagarlickij.com --yes --state s3://kag-kops-state
aws cloudformation delete-stack --stack-name demo4p1
aws cloudformation delete-stack --stack-name demo4p2
aws cloudformation delete-stack --stack-name demo5
aws cloudformation delete-stack --stack-name demo8
```
