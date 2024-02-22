
##################### Create AWS EKS cluster ################################################################################
## Create EKS cluster
eksctl create cluster --name oseks --node-type t2.large --nodes 1 --nodes-min 1 --nodes-max 2 --region ap-south-1

## Get EKS Cluster service
eksctl get cluster --name oseks --region ap-south-1

## Update Kubeconfig 
aws eks update-kubeconfig --name oseks

## Get EKS Pod data.
kubectl get pods --all-namespaces


## Delete EKS cluster
eksctl delete cluster --name oseks --region ap-south-1


###############################################################################################################################
##################### Commands to follows #####################################################################################
## CONFIGURE IRSA FOR FLUENT BIT
-----------------------------------
*Enabling IAM roles for service accounts on your cluster
eksctl utils associate-iam-oidc-provider --cluster oseks --approve

*Creating an IAM role and policy for your service account
#edit resource name in fluent-bit-policy.json
aws iam create-policy --policy-name fluent-bit-policy --policy-document file://fluent-bit-policy.json

* Create an IAM role
kubectl create namespace logging

eksctl create iamserviceaccount --name fluent-bit --namespace logging --cluster oseks --attach-policy-arn "arn:aws:iam::528462248584:policy/fluent-bit-policy" --approve --override-existing-serviceaccounts
	
*Make sure your service account with the ARN of the IAM role is annotated
kubectl -n logging describe sa fluent-bit


##PROVISION AN AMAZON OPENSEARCH CLUSTER
------------------------------------------
# name of our Amazon OpenSearch cluster
ES_DOMAIN_NAME="oseks-logging"

# Elasticsearch version
ES_VERSION="OpenSearch_1.0"

# OpenSearch Dashboards admin user
ES_DOMAIN_USER="oseks"

# OpenSearch Dashboards admin password
ES_DOMAIN_PASSWORD="MysuperSct_Ek1$"

*Create the cluster
aws opensearch create-domain --cli-input-json  file://es_domain.json

*Check health state of es_domain
aws opensearch describe-domain --domain-name oseks-logging

## CONFIGURE AMAZON OPENSEARCH ACCESS
----------------------------------------
* We need to retrieve the Fluent Bit Role ARN
eksctl get iamserviceaccount --cluster oseks --namespace logging -o json

* Get the Amazon OpenSearch Endpoint
aws opensearch describe-domain --domain-name oseks-logging --output text --query "DomainStatus.Endpoint"


https://search-oseks-logging-g22wce3qzhzrt5wxtrxvc7paom.ap-south-1.es.amazonaws.com/_dashboards

* Update the Elasticsearch internal database
curl -sS -u "oseks:MysuperSct_Ek1$" -X PATCH https://search-oseks-logging-g22wce3qzhzrt5wxtrxvc7paom.ap-south-1.es.amazonaws.com/_opendistro/_security/api/rolesmapping/all_access?pretty -H 'Content-Type: application/json' -d '[{"op": "add", "path": "/backend_roles", "value": ["'arn:aws:iam::528462248584:role/eksctl-oseks-addon-iamserviceaccount-logging--Role1-6g5p4VQoQ9EA'"]}]'

## DEPLOY FLUENT BIT
----------------------
*Deploy fluentbit on eks
kubectl apply -f ./fluentbit.yaml

*Check status of the pods
kubectl --namespace=logging get pods



kubectl logs fluent-bit-7pfzr -n logging
## Log in OPENSEARCH
---------------------
*Finally Let’s log into OpenSearch Dashboards to visualize our logs.
OpenSearch Dashboards URL: https://${ES_ENDPOINT}/_dashboards/
OpenSearch Dashboards user: ${ES_DOMAIN_USER}
OpenSearch Dashboards password: ${ES_DOMAIN_PASSWORD}

############################################################################################################################################
