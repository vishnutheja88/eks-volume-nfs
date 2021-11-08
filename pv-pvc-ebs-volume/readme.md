install kubectl
-------------------
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl

update awscli
-------------
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
sudo yum -y install jq gettext bash-completion moreutils

install yq and yaml processing
-------------------------------
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc

Verify the binaries are in the path and executable
-------------------------------------------------
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done

Enable kubectl bash_completion
----------------------------------
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion

set the AWS Load Balancer Controller version
----------------------------------------------
echo 'export LBC_VERSION="v2.3.0"' >>  ~/.bash_profile
.  ~/.bash_profile
---------------------------------------------
kubecluster creation in EKS
---------------------------------------------
eks_workshop.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  #region: us-east-2
  region: ${AWS_REGION}
  version: "1.19"

#availabilityZones: ["us-east-2a", "us-east-2b", "us-east-2c"]
avilabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]
managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.small
  ssh:
    enableSsm: true

# To enable all of the control plane logs, uncomment below:
# cloudWatch:
#  clusterLogging:
#    enableTypes: ["*"]

secretsEncryption:
  keyARN: ${MASTER_ARN}
  #keyARN: arn:aws:kms:us-east-2:268888248929:key/9c02c852-fba2-4c76-9a89-f4b9e0d00960