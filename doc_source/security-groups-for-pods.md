# Security groups for pods<a name="security-groups-for-pods"></a>

Security groups for pods integrate Amazon EC2 security groups with Kubernetes pods\. You can use Amazon EC2 security groups to define rules that allow inbound and outbound network traffic to and from pods that you deploy to nodes running on many Amazon EC2 instance types and Fargate\. For a detailed explanation of this capability, see the [Introducing security groups for pods](http://aws.amazon.com/blogs/containers/introducing-security-groups-for-pods/) blog post\.

**Considerations**

Before deploying security groups for pods, consider the following limits and conditions:
+ Pods running on Amazon EC2 nodes can be in any [Amazon EKS supported Kubernetes version](kubernetes-versions.md), but pods running on [Fargate](fargate.md) must be in a 1\.18 or later cluster\. 
+ Security groups for pods can't be used with Windows nodes\.
+ Security groups for pods can't be used with clusters configured for the IPv6 family that contain Amazon EC2 nodes\. You can however, use security groups for pods with clusters configured for the IPv6 family that contain only Fargate nodes\. For more information, see [Assigning IPv6 addresses to pods and services](cni-ipv6.md)\.
+ Security groups for pods are supported by most [Nitro\-based](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#ec2-nitro-instances) Amazon EC2 instance families, including the `m5`, `c5`, `r5`, `p3`, `m6g`, `c6g`, and `r6g` instance families\. The `t3` instance family is not supported\. For a complete list of supported instances, see the [https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/pkg/aws/vpc/limits.go](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/pkg/aws/vpc/limits.go) file on Github\. Your nodes must be one of the listed instance types that have `IsTrunkingCompatible: true` in that file\.
+ If you're also using pod security policies to restrict access to pod mutation, then the `eks-vpc-resource-controller` and `vpc-resource-controller` Kubernetes service accounts must be specified in the Kubernetes `ClusterRoleBinding` for the `role` that your `psp` is assigned to\. If you're using the [default Amazon EKS `psp`, `role`, and `ClusterRoleBinding`](pod-security-policy.md#default-psp), this is the `eks:podsecuritypolicy:authenticated` `ClusterRoleBinding`\. For example, you add the service accounts to the `subjects:` section, as shown in the following example:

  ```
  ...
  subjects:
    - kind: Group
      apiGroup: rbac.authorization.k8s.io
      name: system:authenticated
    - kind: ServiceAccount
      name: vpc-resource-controller
    - kind: ServiceAccount
      name: eks-vpc-resource-controller
  ```
+ If you're using [custom networking](cni-custom-network.md) and security groups for pods together, the security group specified by security groups for pods is used instead of the security group specified in the `ENIconfig`\.
+ If you're using version 1\.10\.2 or earlier of the Amazon VPC CNI plugin and you include the `terminationGracePeriodSeconds` setting in your pod spec, the value for the setting can't be zero\.  
+ If you're using version 1\.10 or earlier of the Amazon VPC CNI plugin, or version 1\.11 with `POD_SECURITY_GROUP_ENFORCING_MODE`=`strict`, which is the default setting, then Kubernetes services of type `NodePort` and `LoadBalancer` using instance targets with an `externalTrafficPolicy` set to `Local` are not supported with pods that you assign security groups to\. For more information about using a load balancer with instance targets, see [Network load balancing on Amazon EKS](network-load-balancing.md)\. If you're using version 1\.11 or later of the plugin with `POD_SECURITY_GROUP_ENFORCING_MODE`=`standard`, then instance targets with an `externalTrafficPolicy` set to `Local` are supported\.
+ If you're using version 1\.10 or earlier of the Amazon VPC CNI plugin or version 1\.11 with `POD_SECURITY_GROUP_ENFORCING_MODE`=`strict`, which is the default setting, source NAT is disabled for outbound traffic from pods with assigned security groups so that outbound security group rules are applied\. To access the internet, pods with assigned security groups must be launched on nodes that are deployed in a private subnet configured with a NAT gateway or instance\. Pods with assigned security groups deployed to public subnets are not able to access the internet\.

  If you're using version 1\.11 or later of the plugin with `POD_SECURITY_GROUP_ENFORCING_MODE`=`standard`, then pod traffic destined for outside of the VPC is translated to the IP address of the instance's primary network interface\. For this traffic, the rules in the security groups for the primary network interface are used, rather than the rules in the pod's security groups\. 
+ To use Calico network policy with pods that have associated security groups, you must use version 1\.11\.0 or later of the Amazon VPC CNI plugin and set `POD_SECURITY_GROUP_ENFORCING_MODE`=`standard`\. Otherwise, traffic flow to and from pods with associated security groups are not subjected to [Calico network policy](calico.md) enforcement and are limited to Amazon EC2 security group enforcement only\. To update your Amazon VPC CNI version, see [Managing the Amazon VPC CNI add\-on](managing-vpc-cni.md)\.
+ Pods running on Amazon EC2 nodes that use security groups in clusters that use [Nodelocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) are only supported with version 1\.11\.0 or later of the Amazon VPC CNI plugin and with `POD_SECURITY_GROUP_ENFORCING_MODE`=`standard`\. To update your Amazon VPC CNI plugin version, see [Managing the Amazon VPC CNI add\-on](managing-vpc-cni.md)\.

## Configure the Amazon VPC CNI add\-on for security groups for pods<a name="security-groups-pods-deployment"></a>

**To deploy security groups for pods**

If you're using security groups for Fargate pods only, and don't have any Amazon EC2 nodes in your cluster, skip to [Deploy an example application](#sg-pods-example-deployment)\.

1. Check your current CNI plugin version with the following command:

   ```
   kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2
   ```

   Example output:

   ```
   amazon-k8s-cni-init:1.7.5-eksbuild.1
   amazon-k8s-cni:1.7.5-eksbuild.1
   ```

   If your CNI plugin version is earlier than 1\.7\.7, then update your CNI plugin to version 1\.7\.7 or later\. For more information, see [Updating the Amazon VPC CNI Amazon EKS add\-on](managing-vpc-cni.md#updating-vpc-cni-eks-add-on)\.

1. Add the [https://console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonEKSVPCResourceController](https://console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonEKSVPCResourceController) managed policy to the [cluster role](service_IAM_role.md#create-service-role) that is associated with your Amazon EKS cluster\. The policy allows the role to manage network interfaces, their private IP addresses, and their attachment and detachment to and from network instances\.

   1. Determine the ARN of your cluster IAM role\. Replace *my\-cluster* with the name of your cluster\.

      ```
      aws eks describe-cluster --name my-cluster --query cluster.roleArn --output text
      ```

      Output

      ```
      arn:aws:iam::111122223333:role/eksClusterRole
      ```

   1. The following command adds the policy to a cluster role named `eksClusterRole`\. Replace `eksClusterRole` with the name of your role\.

      ```
      aws iam attach-role-policy \
          --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController \
          --role-name eksClusterRole
      ```

1. Enable the Amazon VPC CNI add\-on to manage network interfaces for pods by setting the `ENABLE_POD_ENI` variable to `true` in the `aws-node` `DaemonSet`\. Once this setting is set to `true`, for each node in the cluster the add\-on adds a label with the value `vpc.amazonaws.com/has-trunk-attached=true`\. The VPC resource controller creates and attaches one special network interface called a *trunk network interface* with the description `aws-k8s-trunk-eni`\.

   ```
   kubectl set env daemonset aws-node -n kube-system ENABLE_POD_ENI=true
   ```
**Note**  
The trunk network interface is included in the maximum number of network interfaces supported by the instance type\. For a list of the maximum number of network interfaces supported by each instance type, see [IP addresses per network interface per instance type](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI) in the *Amazon EC2 User Guide for Linux Instances*\. If your node already has the maximum number of standard network interfaces attached to it then the VPC resource controller will reserve a space\. You will have to scale down your running pods enough for the controller to detach and delete a standard network interface, create the trunk network interface, and attach it to the instance\.

   You can see which of your nodes have `aws-k8s-trunk-eni` set to `true` with the following command\. If `No resources found` is returned, then wait several seconds and try again\. The previous step requires restarting the CNI pods, which takes several seconds\.

   ```
   kubectl get nodes -o wide -l vpc.amazonaws.com/has-trunk-attached=true
   ```

   Once the trunk network interface is created, pods are assigned secondary IP addresses from the trunk or standard network interfaces\. The trunk interface is automatically deleted if the node is deleted\. 

   When you deploy a security group for a pod in a later step, the VPC resource controller creates a special network interface called a *branch network interface* with a description of `aws-k8s-branch-eni` and associates the security groups to it\. Branch network interfaces are created in addition to the standard and trunk network interfaces attached to the node\. If you are using liveness or readiness probes, then you also need to disable TCP early demux, so that the `kubelet` can connect to pods on branch network interfaces using TCP\. To disable TCP early demux, run the following command:

   ```
   kubectl patch daemonset aws-node \
     -n kube-system \
     -p '{"spec": {"template": {"spec": {"initContainers": [{"env":[{"name":"DISABLE_TCP_EARLY_DEMUX","value":"true"}],"name":"aws-vpc-cni-init"}]}}}}'
   ```
**Note**  
If you're using 1\.11\.0 or later of the AWS VPC CNI add\-on and set `POD_SECURITY_GROUP_ENFORCING_MODE`=`standard`, as described in the next step, then you don't need to run the previous command\.

1. If your cluster uses NodeLocal DNSCache, or you want to use Calico network policy with your pods that have their own security groups, or you have Kubernetes services of type `NodePort` and `LoadBalancer` using instance targets with an `externalTrafficPolicy` set to `Local` for pods that you want to assign security groups to, then you must be using version 1\.11\.0 or later of the AWS VPC CNI add\-on, and you must enable the following setting:

   ```
   kubectl set env daemonset aws-node -n kube-system POD_SECURITY_GROUP_ENFORCING_MODE=standard
   ```
**Important**  
Pod security group rules aren't applied to traffic between pods or between pods and services, such as `kubelet` or `nodeLocalDNS`, that are on the same node\.
Outbound traffic from pods to addresses outside of the VPC is network address translated to the IP address of the instance's primary network interface \(unless you've also set `AWS_VPC_K8S_CNI_EXTERNALSNAT=true`\)\. For this traffic, the rules in the security groups for the primary network interface are used, rather than the rules in the pod's security groups\.
For this setting to apply to existing pods, you must restart the pods or the nodes that the pods are running on\.

1. Create a namespace to deploy resources to\.

   ```
   kubectl create namespace my-namespace
   ```

1. Deploy an Amazon EKS `SecurityGroupPolicy` to your cluster\.

   1. Save the following example security policy to a file named *my\-security\-group\-policy\.yaml*\. You can replace `podSelector` with `serviceAccountSelector` if you'd rather select pods based on service account labels\. You must specify one selector or the other\. An empty `podSelector` \(example: `podSelector: {}`\) selects all pods in the namespace\. An empty `serviceAccountSelector` selects all service accounts in the namespace\. You must specify 1\-5 security group IDs for `groupIds`\. If you specify more than one ID, then the combination of all the rules in all the security groups are effective for the selected pods\.

      ```
      apiVersion: vpcresources.k8s.aws/v1beta1
      kind: SecurityGroupPolicy
      metadata:
        name: my-security-group-policy
        namespace: my-namespace
      spec:
        podSelector: 
          matchLabels:
            role: my-role
        securityGroups:
          groupIds:
            - sg-abc123
      ```
**Important**  
The security groups that you specify in the policy must exist\. If they don't exist, then, when you deploy a pod that matches the selector, your pod remains stuck in the creation process\. If you describe the pod, you'll see an error message similar to the following one: `An error occurred (InvalidSecurityGroupID.NotFound) when calling the CreateNetworkInterface operation: The securityGroup ID 'sg-abc123' does not exist`\.
The security group must allow inbound communication from the cluster security group \(for `kubelet`\) over any ports you've configured probes for\.
The security group must allow outbound communication to a security group that allows inbound TCP and UDP port 53 \(for CoreDNS pods\) from it\.
If you're using the security group policy with Fargate, make sure that your security group has rules that allow the pods to communicate with the Kubernetes control plane\. The easiest way to do this is to specify the cluster security group as one of the security groups\.
Security group policies only apply to newly scheduled pods\. They do not affect running pods\.

   1. Deploy the policy\.

      ```
      kubectl apply -f my-security-group-policy.yaml
      ```

1. Deploy a sample application with a label that matches the `my-role` value for `podselector` that you specified in the previous step\.

   1. Save the following contents to a file\.

      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-deployment
        namespace: my-namespace
        labels:
          app: my-app
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: my-app
        template:
          metadata:
            labels:
              app: my-app
              role: my-role
          spec:
            containers:
            - name: my-container
              image: my-image
              ports:
              - containerPort: 80
      ```

   1. Deploy the application with the following command\. When you deploy the application, the CNI plugin matches the `role` label and the security groups that you specified in the previous step are applied to the pod\.

      ```
      kubectl apply -f my-security-group-policy.yaml
      ```
**Note**  
If your pod is stuck in the `Waiting` state and you see `Insufficient permissions: Unable to create Elastic Network Interface.` when you describe the pod, confirm that you added the IAM policy to the IAM cluster role in a previous step\.
If your pod is stuck in the `Pending` state, confirm that your node instance type has `IsTrunkingCompatible: true` in [https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/pkg/aws/vpc/limits.go](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/pkg/aws/vpc/limits.go) on Github and that the maximum number of branch network interfaces supported by the instance type multiplied times the number of nodes in your node group hasn't already been met\. For example, an `m5.large` instance supports nine branch network interfaces\. If your node group has five nodes, then a maximum of 45 branch network interfaces can be created for the node group\. The 46th pod that you attempt to deploy will sit in `Pending` state until another pod that has associated security groups is deleted\.

      If you run `kubectl describe pod my-deployment-xxxxxxxxxx-xxxxx -n my-namespace` and see a message similar to the following message, then it can be safely ignored\. This message might appear when the CNI plugin tries to set up host networking and fails while the network interface is being created\. The CNI plugin logs this event until the network interface is created\.

      ```
      Failed to create pod sandbox: rpc error: code = Unknown desc = failed to set up sandbox container "e24268322e55c8185721f52df6493684f6c2c3bf4fd59c9c121fd4cdc894579f" network for pod "my-deployment-59f5f68b58-c89wx": networkPlugin
      cni failed to set up pod "my-deployment-59f5f68b58-c89wx_my-namespace" network: add cmd: failed to assign an IP address to container
      ```

      You cannot exceed the maximum number of pods that can be run on the instance type\. For a list of the maximum number of pods that you can run on each instance type, see [eni\-max\-pods\.txt](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt) on GitHub\. When you delete a pod that has associated security groups, or delete the node that the pod is running on, the VPC resource controller deletes the branch network interface\. If you delete a cluster with pods using pods for security groups, then the controller does not delete the branch network interfaces, so you'll need to delete them yourself\.

## Deploy an example application<a name="sg-pods-example-deployment"></a>

To use security groups for pods, you must have an existing security group and [Deploy an Amazon EKS `SecurityGroupPolicy`](#deploy-securitygrouppolicy) to your cluster, as described in the following procedure\. The following steps show you how to use the security group policy for a pod\. Unless otherwise noted, complete all steps from the same terminal because variables are used in the following steps that don't persist across terminals\.

**To deploy an example pod with a security group**

1. Create a security group to use with your pod\. The steps that follow help you create a simple security group for illustrative purposes only\. In a production cluster, your rules will likely be different\.

   1. Retrieve the IDs of your cluster's VPC and cluster security group\. Replace *my\-cluster* with the name of your cluster\.

      ```
      my_cluster_name=my-cluster
      my_cluster_vpc_id=$(aws eks describe-cluster \
          --name $my_cluster_name \
          --query cluster.resourcesVpcConfig.vpcId \
          --output text)
      my_cluster_security_group_id=$(aws eks describe-cluster \
          --name $my_cluster_name \
          --query cluster.resourcesVpcConfig.clusterSecurityGroupId \
          --output text)
      ```

   1. Create the security group for your pod\. Replace *my\-pod\-security\-group* with your own\. Note the ID of the security group returned in the output after running the commands\. You'll use it in a later step\.

      ```
      my_pod_security_group_name=my-pod-security-group
      aws ec2 create-security-group \
          --vpc-id $my_cluster_vpc_id \
          --group-name $my_pod_security_group_name \
          --description "My pod security group"
      my_pod_security_group_id=$(aws ec2 describe-security-groups \
          --filters Name=group-name,Values=$my_pod_security_group_name \
          --query 'SecurityGroups[].GroupId' \
          --output text)
      echo $my_pod_security_group_id
      ```

   1. Allow TCP and UDP port 53 traffic from the pod security group that you created in the previous step to your cluster security group\. If you want the DNS traffic from your pod to flow to a different security group than your cluster security group, then replace *$my\_cluster\_security\_group\_id* with your own security group ID\. Note the value returned for `SecurityGroupRuleId` in the output returned for each of the commands\. You'll use them in a later step\.

      ```
      aws ec2 authorize-security-group-ingress \
          --group-id $my_cluster_security_group_id \
          --protocol tcp --port 53 --source-group $my_pod_security_group_id
      aws ec2 authorize-security-group-ingress \
          --group-id $my_cluster_security_group_id \
          --protocol udp --port 53 --source-group $my_pod_security_group_id
      ```

   1. Allow inbound traffic to your pod security group from any pod the security group is associated to over any protocol and port\. The security group has a default outbound rule that allows outbound traffic from the pods that your security group is associated with to any destination over any protocol and port\.

      ```
      aws ec2 authorize-security-group-ingress \
          --group-id $my_pod_security_group_id \
          --protocol -1 --port -1 --source-group $my_pod_security_group_id
      ```

1. Create a Kubernetes namespace to deploy resources to\.

   ```
   kubectl create namespace my-namespace
   ```

1. <a name="deploy-securitygrouppolicy"></a>Deploy an Amazon EKS `SecurityGroupPolicy` to your cluster\.

   1. Save the following example security policy to a file named *my\-security\-group\-policy\.yaml*\. You can replace `podSelector` with `serviceAccountSelector` if you'd rather select pods based on service account labels\. You must specify one selector or the other\. An empty `podSelector` \(example: `podSelector: {}`\) selects all pods in the namespace\. An empty `serviceAccountSelector` selects all service accounts in the namespace\. You must specify 1\-5 security group IDs for `groupIds`\. If you specify more than one ID, then the combination of all the rules in all the security groups are effective for the selected pods\. Replace *sg\-05b1d815d1EXAMPLE* with the ID of the security group that you noted in a previous step when you created the security group for your pod\.

      ```
      apiVersion: vpcresources.k8s.aws/v1beta1
      kind: SecurityGroupPolicy
      metadata:
        name: my-security-group-policy
        namespace: my-namespace
      spec:
        podSelector: 
          matchLabels:
            role: my-role
        securityGroups:
          groupIds:
            - sg-05b1d815d1EXAMPLE
      ```
**Important**  
The security group or groups that you specify for your pods must meet the following criteria:  
They must exist\. If they don't exist, then, when you deploy a pod that matches the selector, your pod remains stuck in the creation process\. If you describe the pod, you'll see an error message similar to the following one: `An error occurred (InvalidSecurityGroupID.NotFound) when calling the CreateNetworkInterface operation: The securityGroup ID 'sg-05b1d815d1EXAMPLE' does not exist`\.
They must allow inbound communication from the [cluster security group](sec-group-reqs.md#cluster-sg) \(for `kubelet`\) over any ports that you've configured probes for\.
They must allow outbound communication over TCP and UDP ports 53 to a security group assigned to pods running CoreDNS\. The security group for your CoreDNS pods must allow inbound TCP and UDP port 53 traffic from your security group\.
They must have necessary inbound and outbound rules to communicate with other pods that they need to communicate with\.
They must have rules that allow the pods to communicate with the Kubernetes control plane if you're using the security group with Fargate\. The easiest way to do this is to specify the cluster security group as one of the security groups\.
Security group policies only apply to newly scheduled pods\. They do not affect running pods\.

   1. Deploy the policy\.

      ```
      kubectl apply -f my-security-group-policy.yaml
      ```

1. Deploy a sample application with a label that matches the `my-role` value for `podSelector` that you specified in the previous step\.

   1. Save the following contents to a file named `sample-application.yaml`\.

      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-deployment
        namespace: my-namespace
        labels:
          app: my-app
      spec:
        replicas: 4
        selector:
          matchLabels:
            app: my-app
        template:
          metadata:
            labels:
              app: my-app
              role: my-role
          spec:
            terminationGracePeriodSeconds: 120
            containers:
            - name: nginx
              image: public.ecr.aws/nginx/nginx:1.21
              ports:
              - containerPort: 80
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: my-app
        namespace: my-namespace
        labels:
          app: my-app
      spec:
        selector:
          app: my-app
        ports:
          - protocol: TCP
            port: 80
            targetPort: 80
      ```

   1. Deploy the application with the following command\. When you deploy the application, the CNI plugin matches the `role` label and the security groups that you specified in the previous step are applied to the pod\.

      ```
      kubectl apply -f sample-application.yaml
      ```
**Note**  
If your pod is stuck in the `Waiting` state and you see `Insufficient permissions: Unable to create Elastic Network Interface.` when you describe the pod, confirm that you added the IAM policy to the IAM cluster role in a previous step\.
If your pod is stuck in the `Pending` state, confirm that your node instance type is listed in [https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/pkg/aws/vpc/limits.go](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/pkg/aws/vpc/limits.go) and that the maximum number of branch network interfaces supported by the instance type multiplied times the number of nodes in your node group hasn't already been met\. For example, an `m5.large` instance supports nine branch network interfaces\. If your node group has five nodes, then a maximum of 45 branch network interfaces can be created for the node group\. The 46th pod that you attempt to deploy will sit in `Pending` state until another pod that has associated security groups is deleted\.

      If you run `kubectl describe pod my-deployment-xxxxxxxxxx-xxxxx -n my-namespace` and see a message similar to the following message, then it can be safely ignored\. This message might appear when the CNI plugin tries to set up host networking and fails while the network interface is being created\. The CNI plugin logs this event until the network interface is created\.

      ```
      Failed to create pod sandbox: rpc error: code = Unknown desc = failed to set up sandbox container "e24268322e55c8185721f52df6493684f6c2c3bf4fd59c9c121fd4cdc894579f" network for pod "my-deployment-5df6f7687b-4fbjm": networkPlugin
      cni failed to set up pod "my-deployment-5df6f7687b-4fbjm-c89wx_my-namespace" network: add cmd: failed to assign an IP address to container
      ```

      You can't exceed the maximum number of pods that can be run on the instance type\. For a list of the maximum number of pods that you can run on each instance type, see [eni\-max\-pods\.txt](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt) on GitHub\. When you delete a pod that has associated security groups, or delete the node that the pod is running on, the VPC resource controller deletes the branch network interface\. If you delete a cluster with pods using pods for security groups, then the controller does not delete the branch network interfaces, so you'll need to delete them yourself\.

1. View the pods deployed with the sample application\. For the remainder of this topic, this terminal is referred to as `TerminalA`\.

   ```
   kubectl get pods -n my-namespace -o wide
   ```

   Output

   ```
   NAME                             READY   STATUS    RESTARTS   AGE     IP               NODE                                            NOMINATED NODE   READINESS GATES
   my-deployment-5df6f7687b-4fbjm   1/1     Running   0          7m51s   192.168.53.48    ip-192-168-33-28.region-code.compute.internal   <none>           <none>
   my-deployment-5df6f7687b-j9fl4   1/1     Running   0          7m51s   192.168.70.145   ip-192-168-92-33.region-code.compute.internal   <none>           <none>
   my-deployment-5df6f7687b-rjxcz   1/1     Running   0          7m51s   192.168.73.207   ip-192-168-92-33.region-code.compute.internal   <none>           <none>
   my-deployment-5df6f7687b-zmb42   1/1     Running   0          7m51s   192.168.63.27    ip-192-168-33-28.region-code.compute.internal   <none>           <none>
   ```

1. In a separate terminal, shell into one of the pods\. For the remainder of this topic, this terminal is referred to as `TerminalB`\. Replace **5df6f7687b*\-*4fbjm** with the ID of one of the pods returned in your output from the previous step\.

   ```
   kubectl exec -it -n my-namespace my-deployment-5df6f7687b-4fbjm -- /bin/bash
   ```

1. From the shell in `TerminalB`, confirm that the sample application works\.

   ```
   curl my-app
   ```

   Output

   ```
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ...
   ```

   You received the output because all pods running the application are associated with the security group that you created\. That group contains a rule that allows all traffic between all pods that the security group is associated to\. DNS traffic is allowed outbound from that security group to the cluster security group, which is associated with your nodes\. The nodes are running the CoreDNS pods, which your pods did a name lookup to\.

1. From `TerminalA`, remove the security group rules that allow DNS communication to the cluster security group from your security group\. If you didn't add the DNS rules to the cluster security group in a previous step, then replace *$my\_cluster\_security\_group\_id* with the ID of the security group that you created the rules in\. Replace *sgr\-0407e21efdEXAMPLE* and *sgr\-04dedf9701EXAMPLE* with the IDs of the security group rules that you noted in a previous step\.

   ```
   aws ec2 revoke-security-group-ingress --group-id $my_cluster_security_group_id --security-group-rule-ids sgr-0407e21efdEXAMPLE
   aws ec2 revoke-security-group-ingress --group-id $my_cluster_security_group_id --security-group-rule-ids sgr-04dedf9701EXAMPLE
   ```

1. From `TerminalB`, attempt to access the application again\.

   ```
   curl my-app
   ```

   The attempt fails because the pod is no longer able to access the CoreDNS pods, which have the cluster security group associated to them\. The cluster security group no longer has the security group rules that allow DNS communication from the security group associated to your pod\.

   If you attempt to access the application using the IP addresses returned for one of the pods in a previous step, you still receive a response because all ports are allowed between pods that have the security group associated to them and a name lookup isn't required\.

1. Once you've finished experimenting, you can remove the sample security group policy, application, and security group that you created with the following commands\.

   ```
   kubectl delete namespace my-namespace
   aws ec2 delete-security-group --group-id $my_pod_security_group_id
   ```