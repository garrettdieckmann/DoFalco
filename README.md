# DoFalco
DigitalOcean Kubernetes Challenge 2021

**The Challenge:** Deploy a security and compliance system

# Quick Intro to Falco
Before diving directly into deploying a Kubernetes "runtime security tool", let's first look at two questions:
* what is Kubernetes runtime security?
* what is Falco?

## What is Kubernetes runtime security?
Sysdig, a company with a ["mission ... to make every cloud deployment secure and reliable"](https://sysdig.com/about/) wrote an [article on this exact question](https://sysdig.com/learn-cloud-native/kubernetes-security/runtime-security/), which answers our question as:
* "the protection of containers (or pods) against active threats once the containers are running."
* "Runtime security helps protect workloads against a variety of threats that could emerge after containers have been deployed"

Turns out that the definition of "runtime security" is in the name itself: runtime security being security measures that are taken or performed while the workload is running. For example, runtime security measures may alert on processes that occur in running workloads, such as installation of new operating system packages. 

## What is Falco?
Falco ["is an open source runtime security tool originally built by Sysdig, Inc."](https://falco.org/docs/) As a runtime security tool, Falco listens for events, checks those events against a configurable rules engine and then sends alert messages based upon the pre-defined rules.

## Learning more
To learn more about runtime security and/or Falco, Sysdig has an online training program, complete with an interactive CLI, available online at: https://learn.sysdig.com/falco-101

# Installing Falco
**Note:** The installation docs for the Falco tool recommend installing Falco directly onto the Kubernetes host/node, so that Falco is isolated away from Kubernetes. Additional information and steps for installing directly onto the host are detailed here: https://falco.org/docs/getting-started/installation/. *However*, this guide will demonstrate how to deploy Falco into a Kubernetes cluster using a Helm chart. That Falco Helm chart will deploy Falco as a DaemonSet, so a Falco pod will run on each Kubernetes node. Additionally, as new Kubernetes nodes are added to the cluster, Falco pods will automatically be deployed to the new nodes, and as nodes are removed, the Falco pods on the removed nodes will also be deleted.

## Creating a Kubernetes cluster on DigitalOcean's Kubernetes Service
* Install the `doctl` (https://github.com/digitalocean/doctl) CLI, if not already installed. On a Mac, `doctl` can be installed using Homebrew:
```
> brew install doctl
```
* Authenticate `doctl` with your DigitalOcean account, by providing a personal access token:
```
> doctl auth init

Please authenticate doctl for use with your DigitalOcean account. You can generate a token in the control panel at https://cloud.digitalocean.com/account/api/tokens

Enter your access token:
Validating token... OK
```
* Create a new Kubernetes cluster using the `doctl` CLI (this cluster may be deployed with all of the default Kubernetes cluster options):
```
> doctl kubernetes cluster create falco-demo
Notice: Cluster is provisioning, waiting for cluster to be running
.................................................................................
Notice: Cluster created, fetching credentials
Notice: Adding cluster credentials to kubeconfig file found in "/Users/user/.kube/config"
Notice: Setting current-context to do-nyc1-falco-demo
ID                                      Name          Region    Version        Auto Upgrade    Status     Node Pools
8301075c-c4b4-4223-9494-22f9fb8c66a7    falco-demo    nyc1      1.21.5-do.0    false           running    falco-demo-default-pool
```
**Note:** the `doctl kubernetes cluster create ...` command will, by default, configure the kubeconfig file with the new cluster details, and set the new cluster as the current context. The current context can be found by running: `kubectl config current-context`

## Installing Falco
As noted above, Falco can be deployed through a Helm Chart. For more information on Helm, check out the Helm website: https://helm.sh/
* Add the Falco repository:
```
> helm repo add falcosecurity https://falcosecurity.github.io/charts
"falcosecurity" has been added to your repositories
> helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "falcosecurity" chart repository
Update Complete. ⎈Happy Helming!⎈
```
* Install the Helm chart into our Kubernetes cluster, and into a new `falco` namespace. Also, set `alco.priority=warning` to only load Falco rules that have a minimum rule level of warning.
```
> helm install falco --namespace falco --create-namespace --set falco.priority=warning falcosecurity/falco
NAME: falco
LAST DEPLOYED: Mon Dec 20 10:59:20 2021
NAMESPACE: falco
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Falco agents are spinning up on each node in your cluster. After a few
seconds, they are going to start monitoring your containers looking for
security issues.


No further action should be required.


Tip:
You can easily forward Falco events to Slack, Kafka, AWS Lambda and more with falcosidekick.
Full list of outputs: https://github.com/falcosecurity/charts/falcosidekick.
You can enable its deployment with `--set falcosidekick.enabled=true` or in your values.yaml.
See: https://github.com/falcosecurity/charts/blob/master/falcosidekick/values.yaml for configuration values.
```
* Check to see if the Falco DaemonSet and the Falco pod(s) deployed successfully:
```
> kubectl get daemonset -n falco
NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
falco   3         3         3       3            3           <none>          2m12s
> kubectl get pods -n falco
NAME          READY   STATUS    RESTARTS   AGE
falco-2s92p   1/1     Running   0          3m
falco-bm2rt   1/1     Running   0          3m
falco-x2zwf   1/1     Running   0          3m
```
Above, we see 1 DaemonSet, which manages the Falco pods. We also see 3 Falco pods running, which matches the number of Kubernetes pods in this Kubernetes cluster. The number of Kubernetes nodes in a cluster can be found by running: `kubectl get nodes`.

### Configuring Falco
The Falco Helm chart documentation details the different options that can be passed to the Falco Helm chart: https://github.com/falcosecurity/charts/tree/master/falco#configuration.

Additionally, custom Falco rules can be passed into the running Falco pods even when running Falco via the Helm chart. Instructions on passing custom Falco rules are available in the Falco Helm chart documentation: https://github.com/falcosecurity/charts/tree/master/falco#loading-custom-rules, and additional information about configuring Falco rules is available on the Falco project website: https://falco.org/docs/rules/.

# Falco-in-action
The default Falco deployment through the Helm chart includes a default set of Falco rules, and sends alerts to syslog. Any published alerts will be available in the Falco pod logs.
* Check the logs of all Falco pods (using a label selector):
```
> kubectl logs -l app=falco -n falco
* Trying to load a system falco module, if present
... (snipped)
Mon Dec 20 19:19:39 2021: Falco initialized with configuration file /etc/falco/falco.yaml
Mon Dec 20 19:19:39 2021: Loading rules from file /etc/falco/falco_rules.yaml:
Mon Dec 20 19:19:40 2021: Loading rules from file /etc/falco/falco_rules.local.yaml:
Mon Dec 20 19:19:40 2021: Starting internal webserver, listening on port 8765
```

So far, nothing "interesting" (atleast from Falco's perspective) is occurring in this Kubernetes cluster, and very little is in the Falco pod logs besides some start-up information. Let's trigger a Falco exception! Looking at the default Falco rules, we can see a rule at a priority of 'ERROR' that runs when a package management process is ran inside of a container: https://github.com/falcosecurity/falco/blob/master/rules/falco_rules.yaml#L2422.
* Let's trigger that rule by running a container that updates package information and installs wget:
```
> kubectl run --rm -i -t aptpod --image=ubuntu --restart=Never -- bash -c "apt update && apt install wget -y"
Unpacking libpsl5:amd64 (0.21.0-1ubuntu1) ...
... (snipped)
done.
pod "aptpod" deleted
```

Great! The `aptpod` pod ran, updated its package source information and successfully installed wget.
* Checking the Falco logs again (and grepp'ing for 'Error'), we see that Falco received the system call that a package manager was running, generated an alert and sent that alert to the pod's syslog:
```
> kubectl logs -l app=falco -n falco | grep 'Error'
19:41:11.987796765: Error Package management process launched in container (user=root user_loginuid=-1 command=apt update container_id=4fb52f08328d container_name=<NA> image=<NA>:<NA>) k8s.ns=<NA> k8s.pod=<NA> container=4fb52f08328d k8s.ns=<NA> k8s.pod=<NA> container=4fb52f08328d
19:41:17.410948991: Error Package management process launched in container (user=root user_loginuid=-1 command=apt install wget -y container_id=4fb52f08328d container_name=aptpod image=docker.io/library/ubuntu:latest) k8s.ns=default k8s.pod=aptpod container=4fb52f08328d k8s.ns=default k8s.pod=aptpod container=4fb52f08328d
```
In the Falco logs, we can see:
* a log line for each command, one for the `apt update` and one for the `apt install wget -y`
* container metadata - the user name, the command, the container id and even the Docker image
* pod metadata - which namespace this pod was runnign in, and the pod name

**Note:** Recall that Falco receives system calls from the host it is running on, and that the default Falco configuration just logs directly to the pod's Syslog. Given that, alerts will only be logged on the Falco pod that runs on the same Kubernetes node/host where the Falco rule was triggered; the alert will not execute, or be logged on the Falco pods that are on different nodes than the node our `aptpod` pod ran on.

### Additional Falco alert channels
As mentioned above, the default Falco configuration only sends alerts to the pod's Syslog. Falco also has additional alert channels, including:
* Standard Output
* File
* Syslog
* Spawn a program
* HTTP(s) endpoint
* gRPC API call

which are covered in more detail here: https://falco.org/docs/alerts/

# Cleaning up
* To clean-up the resources created in this guide, but preserve the Kubernetes cluster, uninstall the helm release:
```
> helm uninstall falco -n falco
release "falco" uninstalled
```
* To delete everything, delete the entire Kubernetes cluster:
```
> doctl kubernetes cluster delete falco-demo
Warning: Are you sure you want to delete this Kubernetes cluster? (y/N) ? y
Notice: Cluster deleted, removing credentials
Notice: Removing cluster credentials from kubeconfig file found in "/Users/user/.kube/config"
Notice: The removed cluster was set as the current context in kubectl. Run `kubectl config get-contexts` to see a list of other contexts you can use, and `kubectl config set-context` to specify a new one.
```
