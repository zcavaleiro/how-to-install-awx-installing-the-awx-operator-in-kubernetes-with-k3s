# How to install AWX. Installing the AWX Operator in Kubernetes with K3s


## Table of contents
- [Objective](#objective)
- [What is AWX?](#what-is-awx)
- [Kubernetes frameworks](#kubernetes-frameworks)
- [Installing K3s](#installing-k3s)
- [AWX Operator Installation](#awx-operator-installation)
  - [Installing Kustomize](#installing-kustomize)
  - [Installing the AWX Operator](#installing-the-awx-operator)
  - [Your credentials, password for AWX](#your-credentials-password-for-awx)
- [Bibliography](#bibliography)
  - [Docs](#docs)



<br>

## Objective
 - Deploying or installing AWX has changed a lot since recent years. For the less familiarized with containers and kubernetes it can be a challenging task.
 - This guide aims to help with that, providing a basic installation of AWX for local development and testing purposes.
 - Operating system used:
   - Virtual machine (vm) with 4 cores, 6Gb of Ram and 30Gb of disk size.
   - Ubuntu server LTS 22.04.05.

## What is AWX?
[AWX](https://www.ansible.com/awx/) is an open-source project that provides a graphical user interface for IT infrastructure management and automation with Ansible in mind. Has the support from Redhat and is the “spin-off” of the commercial version of Ansible Tower, part of the Redhat Automation Platform. AWX is the *go to* application for managing your ansible playbooks, but can also work with pyhton, bash scripts, terraform, etc.
For more information visit the [AWX github repo](https://github.com/ansible/awx).  
Another similar tool, for context, is [Semaphoreui](https://semaphoreui.com/). From their homepage, “Effortlessly manage the tasks with a modern, intuitive interface built for DevOps teams.”


<br>

## Kubernetes frameworks:
- One of the recommended methods for installing AWX is with Kubernetes (K8s). We have to take care of that first and then install the AWX operator. By the way, credits to the author of the operator, [Jeef Geerling](https://www.jeffgeerling.com/).
- Kubernetes, as a container manager ,requires hardware resources that are often difficult to gather for testing environments on portable computers or even personal computers.
- In order for developers to be able to use Kubernetes with fewer hardware resources and quick setup, the following frameworks have emerged:
  - Minikiube is often suggested as an alternative environment but is only recommended for development purposes.
  - [K3s](https://k3s.io/), is Kubernetes certified, is more stable than minikube and will be our solution.


<br>

## Installing K3s
After updating Ubuntu, to install K3s:

```sh
curl -sfL https://get.k3s.io | sh -
```

  - The `kubectl --version` command has been replaced by `kubectl version`.
  - However, when running with the normal user it will give an "error" due to lack of permissions.
By running `sudo kubectl version` it is now possible to see version information. Let's change the ownership of the file in question.
  - Check file permissions:

```sh
ls -lsa /etc/rancher/k3s/k3s.yaml
#
# 4 -rw------- 1 root root 2661 jul 23 12:49 /etc/rancher/k3s/k3s.yaml
#
```

- We can verify that only root has permissions.

- To change permissions, let's change ownership for our user and group.

```sh
# to find out our username:
whoami

# to find out our groups:
groups

# to change permissions
# sudo chown username:group /etc/rancher/k3s/k3s.yaml
sudo chown ana:ana /etc/rancher/k3s/k3s.yaml
```

- This way the kubectl version command can now be executed by the user.

Now we can see the active nodes:
```sh
kubectl get nodes
#NAME STATUS ROLES AGE VERSION
#ubuntu-awx Ready control-plane,master 52m v1.25.4+k3s1
```

If we look for the pods:
```sh
kubectl get pods
#No resources found in default namespace.
```
- `default` is the standard name assigned to a namespace.
- For different services, isolation, etc., we can create different namespaces, in our case it will be **awx**.


<br>

## AWX Operator Installation

### Installing Kustomize
One of the ways to install and configure AWX Operator, according to the [documentation](https://ansible.readthedocs.io/projects/awx-operator/en/latest/installation/basic-install.html, is with the help of the [Kustomize](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/) program. Let's opt for the binary installation method:

```sh
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```
- After downloading, we have to move the binary to be "accessible" to the user.

```sh
sudo mv kustomize /usr/local/bin
```
Confirmation check:
```sh
which kustomize
#/usr/local/bin/kustomize
```

<br>

### Installing the AWX Operator

To start the installation, we need to create a file named `kustomization.yml`.

```sh
sudo vim kustomization.yml
```
With the following content:
```sh
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
 - github.com/ansible/awx-operator/config/default?ref=2.19.1
images:
 - name: quay.io/ansible/awx-operator
 newTag: 2.19.1
namespace: awx
```

After saving the file, let's build it to start the installation:

```sh
kustomize build . | kubectl apply -f -
```

Lets looks for the pods in our namespace:
```sh
kubectl get pods --namespace awx
#NAME                                           READY STATUS RESTARTS
AGE
#awx-operator-controller-manager-656f9859-y4zlf 2/2 Running 0
3m39s
```
- When the READY field is 2/2, the awx-operator resources are ready.

Now we need create another `awx.yml` file with the content:
```sh
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
 name: awx
spec:
 service_type: nodeport
 nodeport_port: 30800
```
In the file created previously, `kustomize.yml`, we will add a line to take these `awx.yml` resource settings into account.   

Lets change again our `kustomize.yml`, to make it like:

```sh
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
 - github.com/ansible/awx-operator/config/default?ref=2.19.1
 - awx.yml
images:
 - name: quay.io/ansible/awx-operator
 newTag: 2.19.1
namespace: awx
```

By making a new build, like the previous `build` command, **AWX** will have more resources and those it had before will remain unchanged.
```sh
kustomize build . | kubectl apply -f -
#namespace/awx unchanged
#customresourcedefinition.apiextensions.k8s.io/awxbackups.awx.ansible.com unchanged
#customresourcedefinition.apiextensions.k8s.io/awxrestores.awx.ansible.com unchanged
#customresourcedefinition.apiextensions.k8s.io/awxs.awx.ansible.com unchanged
#serviceaccount/awx-operator-controller-manager unchanged
#role.rbac.authorization.k8s.io/awx-operator-awx-manager-role configured
#role.rbac.authorization.k8s.io/awx-operator-leader-election-role unchanged
#clusterrole.rbac.authorization.k8s.io/awx-operator-metrics-reader unchanged
#clusterrole.rbac.authorization.k8s.io/awx-operator-proxy-role unchanged
#rolebinding.rbac.authorization.k8s.io/awx-operator-awx-manager-rolebinding unchanged
#rolebinding.rbac.authorization.k8s.io/awx-operator-leader-election-rolebinding unchanged
#clusterrolebinding.rbac.authorization.k8s.io/awx-operator-proxy-rolebinding unchanged
#configmap/awx-operator-awx-manager-config unchanged
#service/awx-operator-controller-manager-metrics-service unchanged
#deployment.apps/awx-operator-controller-manager configured
#awx.awx.ansible.com/awx created
```
- This step will take time to configure and install. We can follow the installation log output in a separate ssh window/session with the command:

    ```sh
    kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager -- namespace awx
    ```
    - In the end, we will have the PLAY RECAP in which we must have the status of failed=0 and unreachable=0.

In order to verify that we have the deployment done, we can check the pods:
```sh
kubectl get pods -awx
```

We should have an outputput similar to:
```sh
NAME                                            READY   STATUS  RESTARTS
AGE
awx-7497bc89d7-nxlnw                            4/4     Running 408 (8m15s ago) 15h
awx-postgres-13-0                               1/1     Running 2 (8m15s ago) 15h
awx-operator-controller-manager-9589d9859-x2zlf 2/2     Running 5 (8m15s ago) 16h

```


<br>

### Your credentials, password for AWX

Finally, to login into the awx web application we need the automatically generated password. By default, the `admin` account and password are in `<resourcename>-admin-password`. To check the secrets:
```sh
kubectl get secrets -n awx
```

To check the password:
```sh
kubectl -n awx get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode
```
The string that appears in the below format. Up to the user name Ana, is our initial password:
```sh
U1qizJRANDOMPASSNQAZlnNDk9QAna@ubuntu-awx:~$
```

Within the same network on a machine with a browser, we can now test the access through the vm's IP on the defined port, 30800.

<br>

## Bibliography

### Docs
  - https://ansible.readthedocs.io/projects/awx-operator/en/latest
  - https://docs.k3s.io/quick-start
  - https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/


<br>

<div dir=RtL> 
:Author</br>
<i>Zito Cavaleiro  </i>-   email: zcavaleiro AT protonmail DOT com</br>
Mathematics and Natural Sciences Teacher</br>
IT Automation Engineer</div>
