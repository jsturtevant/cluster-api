---
title: Windows Support
authors:
  - "@jsturtevant"
  - "@ksubrmnn"
reviewers:
  - "@CecileRobertMichon"
  - "@ncdc"
creation-date: 2020-08-25
last-updated: 2020-09-09
status: implementable
see-also:
---

# Windows Support

## Table of Contents

- [Windows Support](#windows-support)
  - [Table of Contents](#table-of-contents)
  - [Glossary](#glossary)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals/Future Work](#non-goalsfuture-work)
  - [Proposal](#proposal)
    - [Cluster API Bootstrap Provider Kubeadm](#cluster-api-bootstrap-provider-kubeadm)
      - [cloud-init and cloudbase-init](#cloud-init-and-cloudbase-init)
      - [Image Creation](#image-creation)
      - [Kubelet and other component configuration](#kubelet-and-other-component-configuration)
      - [netbios names](#netbios-names)
    - [Infrastructure provider implementation](#infrastructure-provider-implementation)
    - [User Stories](#user-stories)
      - [As an operator, I would like to create Windows OS worker nodes with the CAPI API.](#as-an-operator-i-would-like-to-create-windows-os-worker-nodes-with-the-capi-api)
      - [As an operator, I would like to manage Windows OS worker nodes with the CAPI API.](#as-an-operator-i-would-like-to-manage-windows-os-worker-nodes-with-the-capi-api)
    - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
      - [Signing of the components.](#signing-of-the-components)
      - [Known prototypes and prior work:](#known-prototypes-and-prior-work)
    - [Security Model](#security-model)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Alternatives](#alternatives)
  - [Upgrade Strategy](#upgrade-strategy)
  - [Additional Details](#additional-details)
    - [Test Plan [optional]](#test-plan-optional)
    - [Graduation Criteria [optional]](#graduation-criteria-optional)
      - [Alpha](#alpha)
      - [Beta](#beta)
      - [Stable](#stable)
    - [Version Skew Strategy](#version-skew-strategy)
  - [Implementation History](#implementation-history)

## Glossary

Refer to the [Cluster API Book Glossary](https://cluster-api.sigs.k8s.io/reference/glossary.html).

## Summary

This proposal is for the support of the Windows nodes with CAPI and [infrastructure providers](https://cluster-api.sigs.k8s.io/reference/glossary.html#infrastructure-provider) that wish to support Windows [OS](https://cluster-api.sigs.k8s.io/reference/glossary.html#operating-system) worker nodes.  
Windows has been Stable in Kubernetes since 1.14 and is supported in mix worker node scenarios.  

Windows support will provide the ability to add Windows nodes to a [workload cluster](https://cluster-api.sigs.k8s.io/reference/glossary.html#workload-cluster).  Windows nodes are configured differently across 
solutions and the centralization of the scripts and guidance in the Cluster API solution will provide 
guidance for the community on best practices for configuring Windows nodes at scale.

Windows node support has some unique challenges because of the current limitations of Windows Containers. 
Windows containers do not support privileged operations which means that configuration and access to the host
machine must be done at provisioning time.  

An example of this limitation is how kube-proxy gets configured on Windows nodes. Kube-proxy typically run as a 
Windows service on the host machine and it cannot be deployed as a Daemon set as it is on Linux.  
To address this limitation the community has built tools such as the [CSI-Proxy](https://github.com/kubernetes-csi/csi-proxy), which is a CSI driver specific proxy. This proposal will address how to approach the configuration of components 
that are typically deployed as daemon sets when bootstrapping Windows nodes in CAPI.

## Motivation
Kubernetes has supported Windows workloads since the release of Windows support in Kubernetes 1.14. The motivation of this project is to enable Cluster API customers to deploy Windows as part of a mixed OS cluster via the Cluster API automation built for platform operators.  This will enable cluster operators to define Windows machines in the same consistent and repeatable fashion.

### Goals

- Enable the creation and management of Windows worker nodes on workload clusters by adding support via the Kubeadm bootstrap provider and infrastructure providers
- Provide community guidance and scripts for building base images for Windows nodes
- Re-use of the existing Cluster API Bootstrap Provider Kubeadm and other tools where appropriate

### Non-Goals/Future Work

- Provide a way to run [control plane](https://cluster-api.sigs.k8s.io/reference/glossary.html#control-plane) nodes as Windows
- Support for Windows versions outside of the Kubernetes support versions
- Support for Windows nodes on the [management](https://cluster-api.sigs.k8s.io/reference/glossary.html#management-cluster) or [bootstrap clusters](https://cluster-api.sigs.k8s.io/reference/glossary.html#bootstrap-cluster)
- Provide a way to configure Windows nodes with non-Kubeadm based bootstrap providers

## Proposal

There are several components that need to be addressed for Windows support:

- [Cluster API Bootstrap Provider Kubeadm](https://github.com/kubernetes-sigs/cluster-api/tree/master/bootstrap/kubeadm)
- Image creation 
- Kubelet and other component configuration
- Infrastructure provider implementation 

### Cluster API Bootstrap Provider Kubeadm

#### cloud-init and cloudbase-init
For Linux, when using the Kubeadm bootstrap provider, the bootstrap script is provided to the infrastructure provider as a cloud-init script. 
The infrastructure provider is responsible for putting the cloud-init script in the right location. 
When the VM is booted, the cloud-init script runs automatically. 

Cloud-init does not have Windows support. An alternative product is [cloudbase-init](https://github.com/cloudbase/cloudbase-init). 
Cloudbase-init functions in the same way as cloud-init and can consume cloud-init scripts as provided by the Cluster API Bootstrap Provider Kubeadm.  
By using cloudbase-init, Windows can leverage the existing solutions and stay up to date with the latest changes in CABPK.

#### Image Creation
Using cloudbase-init requires the creation of a image with the tooling installed on it since it is not 
provided out of the box by any cloud providers.  To address this we will provide packer scripts as part of 
the [image-builder project](https://github.com/kubernetes-sigs/image-builder) that will pre-install 
cloudbase init.  It is important to note that while scripts can be provided to build an image, all images 
built will need to adhere to [Windows licensing requirements](https://www.microsoft.com/en-us/licensing/product-licensing/windows-server).

There is prior art for building Windows base images. For example, AKS-Engine has an example implementation for using packer and scripts to do image configuration: https://github.com/Azure/aks-engine/blob/master/vhd/packer/windows-vhd-builder.json.  
Another example is the the [sig-windows-tools]([sig-windows-tools](https://github.com/kubernetes-sigs/sig-windows-tools)) which provide scripts for image configuration when using Kubeadm.

Although the Linux implementation in image-builder uses Ansible for configuration, Windows will not share
the same configuration because [Ansible](https://docs.ansible.com/ansible/latest/user_guide/windows.html) requires [Windows specific modules](https://docs.ansible.com/ansible/latest/modules/list_of_windows_modules.html) to do the configuration. 

#### Kubelet and other component configuration
Due to the lack of privileged containers in Windows, a combination of `PreKubeadmCommands`/`PostKubeadmCommands` 
scripts and wins.exe can be used to configure the nodes. Wins.exe is currently provided as a way to bootstrap nodes along with kubeadm in the [Kubernetes documentation for adding Windows nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-windows-nodes/).  
The components from the [preparenode script](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-windows-nodes/#joining-a-windows-worker-node) can be used during image creation.

In the future with the introduction of the [Privileged Containers for Windows containers](https://github.com/kubernetes/enhancements/issues/1981) we can use Privileged containers in place of wins.exe enabled containers. 
As the privileged container support is still being discussed, the replacement of the `PreKubeadmCommands`/`PostKubeadmCommands` kubeadm scripts might be possible but will need to be addressed at a later point in time. 

The plan during Alpha implementation is that each of the infrastructure providers will provide their own `PreKubeadmCommands`/`PostKubeadmCommands` scripts. 
For Beta we should be able to identify common overlapping features that can be added directly to the kubeadm cluster provisioning 
(CABPK) in a similar way that the [kubeadm-bootstrap-script.sh](https://github.com/kubernetes-sigs/cluster-api/blob/master/bootstrap/kubeadm/internal/cloudinit/kubeadm-bootstrap-script.sh) is done today.

#### netbios names
Netbios name resolution concerns were addressed in https://github.com/kubernetes-sigs/cluster-api/issues/2217#issuecomment-598999436:

> If the DNS suffix search order is properly populated in Windows nodes, the long host names Cluster-API generates should directly be usable. And whether NETBIOS is enable or not shouldn't matter. 

### Infrastructure provider implementation
By leveraging cloudbase-init, an infrastructure provider implementation will require only a few changes which include:

- Make changes to their provider api to enable Windows OS infra machines
- Ensuring the cloudbase-init data lands in the correct place on the machines for Windows OS worker nodes

From the infrastructure provider perspective, there are no known required changes to the CAPI API to support Windows
nodes at this time.  If during alpha we identify changes we will open issues pertaining to the changes required.

### User Stories

#### As an operator, I would like to create Windows OS worker nodes with the CAPI API.

#### As an operator, I would like to manage Windows OS worker nodes with the CAPI API.

### Implementation Details/Notes/Constraints

Due to the lack of privileged containers in Windows there are two options for configuring the components such as 
kube-proxy, kubelet.  The above solution using wins is preferred because it makes the move to privileged containers 
straightforward as a drop and replace. 

While this is the best choice for the alpha and the community direction there are some infrastructure providers that may 
not be able to use wins due to signing or security concerns since wins allows the execution of any arbitrary command on 
the host. Pre/post commands can be used as an alternative with additional scripts cached on the image that enable the 
configuration.

#### Signing of the components.  
Some infrastructure providers will require any scripts and binaries are signed before deployment.  
This will be managed by providing the ability to provide url's to override external scripts and binaries during the image building process. An example of how this is could be accomplished is in the Linux implementation is the [containerd_url](https://github.com/kubernetes-sigs/image-builder/blob/58a08a1a8241356bab4afb1c6d8d2fbb8ef54bcf/images/capi/packer/config/ansible-args.json).  In this case, the `containerd_url` could point to a location that would contain a packaged with signed binaries from the infrastructure provider.

#### Known prototypes and prior work: 
- https://github.com/adelina-t/cloudbase-init-capz-demo
- https://github.com/benmoss/kubeadm-windows/tree/master/cluster-api-aws
- https://github.com/microsoft/cluster-api-provider-azurestackhci

### Security Model

Wins.exe is currently the [recommended way to use kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-windows-nodes/).  
Limiting access to the named pipes that are required is one way to mitigate access.  Wins is currently 
required during and after provisioning for running kube-proxy and the CNI daemonset.  

The security model for Privileged containers is still being discussed in KEP and is still early.  The security concerns for Privileged containers will be addresses in the Beta phase of this proposal after the Privileged containers KEP progresses.

Kubeadm bootstrap token should be able to use multi-part mime documents for cloudbase-init as done for [Linux in CAPA](https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/28d01d064cc2e5b0286ae23b3be7203f18b00447/controllers/awsmachine_controller.go#L601). 
This will require an update to Cloudbase-init which does support [mutli-part mime documents](https://cloudbase-init.readthedocs.io/en/latest/userdata.html#multi-part-content) but is missing the [boothook](https://cloudinit.readthedocs.io/en/latest/topics/format.html?highlight=boothook#cloud-boothook) functionality.  
Support for cloudhooks can be added to cloudbase-init to meet the AWS provider requirement that 
would bring parity to Linux implementation during the Beta phase.

There is [no known requirement](https://github.com/kubernetes-sigs/cluster-api/issues/2218) for managing the Admin
Kubeconfig or domain passwords in the Windows configuration. Domain passwords should be managed outside the scope of 
CAPI and only Kubeadm bootstrap tokens, which have limited lifetime, should be used for the joining the Windows nodes. 

### Risks and Mitigations

- Privileged containers are not implemented.
  - There is an active discussion and [KEP](https://docs.google.com/document/d/12EUtMdWFxhTCfFrqhlBGWV70MkZZPOgxw0X-LTR0VAo/edit#) in place.  At the Beta stage the community can do a checkpoint to determine if the solution fits customer needs
- Cloudbase-init is a third party dependency
  - This project is under Apache 2.0 License : https://github.com/cloudbase/cloudbase-init which is cleared under the CNCF Allow list: https://github.com/cncf/foundation/blob/master/allowed-third-party-license-policy.md
- Windows image Distribution
  - Infrastructure providers can provide the ability to use customer provided images and images provided by image-promoter are recommended for testing and demonstration purposes. It is recommended the customer creates their own image. 
  - Customers using the image scripts must ensure they are following [Windows licensing requirements](https://www.microsoft.com/en-us/licensing/product-licensing/windows-server)
- Wins.exe is a third party dependency
  - The project is under the Apache 2.0 License

## Alternatives

1. An alternative to using wins and daemonsets to do the configuration to download and configure components as services 
   using the kubeadm pre/post commands. This would require the infrastructure providers to have the ability to pass 
   configuration through the use of these commands which is already done today. During the Alpha phase with the pre/post 
   scripts being developed by individual infra providers this will not be an issue.  With the move to Windows privileged
   containers in Beta, this becomes a non issue as wins will no longer be required.

1. Create a separate bootstrap provider for Windows.  This would require re-implementing a lot of the logic that is 
   already in CABPK.  
   When bugs are fixed or changes in behavior occur, Windows would risk being out of sync with the Linux implementation.

1. Modified CABPK provider to have a different output format than cloud-init for Windows nodes.   With Cloudbase-init 
   there are no requirements to change CABPK which makes the adaption for Windows straightforward. If we were to adapt 
   the output format of CABPK there is a potential for introduction of bugs and variation in the logic that would be 
   created for Windows nodes.  This would cause the Windows implementation to differ from others which could lead to 
   confusion when debugging differences.

## Upgrade Strategy

Nodes that use this pattern will require the infrastructure to be immutable as specified by CAPI documentation.

## Additional Details

### Test Plan [optional]

**Note:** *Section not required until targeted at a release.*

Given there are no changes to the CAPI codebase, The testing plan would be left up to each infrastructure provider. 
It would be recommended to leverage the existing upstream Windows tests to show that Windows nodes are operating 
effectively.  If changes are required to the CAPI codebase unit tests and if appropriate e2e tests should be added.

### Graduation Criteria [optional]

**Note:** *Section not required until targeted at a release.*

#### Alpha
`PreKubeadmCommands`/`PostKubeadmCommands` command scripts are created per infrastructure provider
Initially implemented with wins and pre/post commands
Windows Packer script to create image with cloudbase-init added to the image builder scripts

#### Beta
- Pre/Post commands moved to bootstrap provider
- Adopt privileged containers (dependant on Privileged containers KEP)
- kubeadm bootstrap token can be kept secret via multi-part mime documents for cloudbase-init. 

#### Stable
Use of privileged containers. 

### Version Skew Strategy 

The version of support for the Windows operating system is outside the scope of cluster creation.  Please refer 
to the Kubernetes Windows documentation for the latest version skew support, features, and functionality.

## Implementation History

- [X] 01/29/2020: Proposed idea in an issue
  - https://github.com/kubernetes-sigs/cluster-api/issues/2218
  - https://github.com/kubernetes-sigs/cluster-api-provider-azure/issues/153
- [X] 08/17/2020: Compile a Google Doc following the CAEP template 
  - https://docs.google.com/document/d/14evDl_3RgEFfchmgPzNw6lb1vIttN_Hb9333UnUJ734/edit
- [X] 08/31/2020: First round of feedback from community
- [X] 08/25/2020: Present proposal at a [community meeting]
- [X] 09/09/2020: Open proposal PR

<!-- Links -->
[community meeting]: https://docs.google.com/document/d/1fQNlqsDkvEggWFi51GVxOglL2P1Bvo2JhZlMhm2d-Co/edit#heading=h.ozawn3ogj91o

