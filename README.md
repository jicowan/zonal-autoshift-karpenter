# zonal-autoshift-karpenter
This is a concept showing how you can use zonal autoshift's integration with EventBridge to automatically remove zones from a Karpenter node pool when an autoshift is underway.

## High level setup instructions
1. Create an EventBridge rule for zonal shift, "Autoshift in progress" events.
2. Configure the rule to send autoshift events to an SNS topic.
3. Create an IAM role for the zonal-autoshift-karpenter pod to assume that allows it to subscribe to the topic. If using IRSA, add the roleArn to the pod's service account. If using Pod Identity, create a pod identity association. 
4. Apply the zonal-autoshift-karpenter deployment.yaml

## TODOs
1. If no topology.kubernetes.io/zone key exists, create it, then add the list of unimpared zones as values. Use DescribeCluster to get the current list of eligible subnets/availability zones. If it's an auto-mode cluster and there are no custom node pools, create a new node pool from the general purpose node pool and add the topology.kubeberes.io/zone key and the list of unimpared zones as values.
2. When a zonal shift occurs, and if the topology.kubernetes.io/zone key exists, store the name of the impared zone, e.g. us-west-2a, as an annotation in the node pool before removing it from the list of values. When service is restored (the zonal shift is cancelled or expires), restore the zone to the list of values by retrieving it from the annotations. Alternatively, we can get the list of zones from the VPC subnets that have the Karpenter autodiscover tag and restore the missing zone. See [this issue](https://github.com/jicowan/zonal-autoshift-karpenter/issues/1) for further discussion on this topic. 
3. Secure the HTTP endpoint for the SNS client that runs in the k8s cluster or, if that's not possible, switch to SQS.
4. Use an Infrastructure as Code tool such as TF or CDK to automate the deployment.

## Other considerations
1. Does the application need to implement leader election? If the pod subscribed to the SNS topic is running on an instance in an impaired zone, it may not receive the event. Avoiding this issue will require multiple replicas spread across different (at least 2 AZs). If multiple pods are subscribed to the topic, they will all try to update the node pool when they receive the event that a zonal shift has occurred. With leader election, if the primary pod is "dead" another pod in the deployment will become the new leader. The application would need to be written such that only the leader could update the node pool.  
2. Should this be implemented outside of Kubernetes? If you implement this application using Lambda instead of Kubernetes you no longer have a dependency on Kubernetes, however, you will need to deploy the Lambda in a VPC because there's no guarantee that the cluster API endpoint will be public. Lambda executions may also be impacted if the compute used for running the function is in the impaired zone. You would also need to grant the execution role for the function access to update node pools (using access management or updating the aws-auth ConfigMap whereas with pods you only need to create a service account and cluster role/binding).
3. How is SNS impacted by a zone impairment?
