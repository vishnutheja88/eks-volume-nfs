# eks-volume-nfs
1. namespace
2. service account - nfs-clietn-provisioner
3. clusterroles: rbac 
4. cluster role binding: rbac-bind
5. Role
6. RoleBinding
7. deployment
8. storage class
9. pv
10. pvc
*******************************************
step 1: create IAM OIDC provider
$eksctl utils associate-iam-oidc-provider --region ${AWS_REGION} --cluster eksworkshop-eksctl --approve

stpe 2: create IAM policy called
$curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.0/docs/install/iam_policy.json
$aws iam create_policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

step 3: creat IAM role and ServiceAccount
$ eksctl create iamserviceaccount --cluster eksworkshop-eksctl --namespace kube-system --name aws-load-balancer-controller --attach-policy-arn arn:aws:iam:${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve

step 4: TargetGroupBinding CRDs
$kubectl apply -k github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master
$kubectl get crd

step 5: Helm Chart
$helm repo add eks https://aws.github.io/eks-charts

$helm upgrade -i aws-load-balancer-controller \
    eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=eksworkshop-eksctl \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set image.tag="${LBC_VERSION}"

$kubectl -n kube-system rollout status deployment aws-load-balancer-controller

STEP 6: Deploy sample Application
$export EKS_CLUSTER_VERSION=$(aws eks describe-cluster --name eksworkshop-eksctl --query cluster.version --output text)

if [ "`echo "${EKS_CLUSTER_VERSION} < 1.19" | bc`" -eq 1 ]; then     
    curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full.yaml \
    | sed 's=alb.ingress.kubernetes.io/target-type: ip=alb.ingress.kubernetes.io/target-type: instance=g' \
    | kubectl apply -f -
fi

if [ "`echo "${EKS_CLUSTER_VERSION} >= 1.19" | bc`" -eq 1 ]; then     
    curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full_latest.yaml \
    | sed 's=alb.ingress.kubernetes.io/target-type: ip=alb.ingress.kubernetes.io/target-type: instance=g' \
    | kubectl apply -f -
fi

$kubectl get ingress/ingress-2048 -n game-2048
$export GAME_INGRESS_NAME=$(kubectl -n game-2048 get targetgroupbindings -o jsonpath='{.items[].metadata.name}')

$kubectl -n game-2048 get targetgroupbindings ${GAME_INGRESS_NAME} -o yaml

$export GAME_2048=$(kubectl get ingress/ingress-2048 -n game-2048 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

$echo http://${GAME_2048}

*********************************************************
clean up the eks cluster
*********************************************************
$kubectl delete -f ~/environment/run-my-nginx.yaml
$kubectl delete ns my-nginx
$rm ~/environment/run-my-nginx.yaml

$export EKS_CLUSTER_VERSION=$(aws eks describe-cluster --name eksworkshop-eksctl --query cluster.version --output text)

if [ "`echo "${EKS_CLUSTER_VERSION} < 1.19" | bc`" -eq 1 ]; then     
    curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full.yaml \
    | sed 's=alb.ingress.kubernetes.io/target-type: ip=alb.ingress.kubernetes.io/target-type: instance=g' \
    | kubectl delete -f -
fi

if [ "`echo "${EKS_CLUSTER_VERSION} >= 1.19" | bc`" -eq 1 ]; then     
    curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full_latest.yaml \
    | sed 's=alb.ingress.kubernetes.io/target-type: ip=alb.ingress.kubernetes.io/target-type: instance=g' \
    | kubectl delete -f -
fi

unset EKS_CLUSTER_VERSION

$helm uninstall aws-load-balancer-controller \
    -n kube-system

$kubectl delete -k github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master

$eksctl delete iamserviceaccount \
    --cluster eksworkshop-eksctl \
    --name aws-load-balancer-controller \
    --namespace kube-system \
    --wait

$aws iam delete-policy \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy
