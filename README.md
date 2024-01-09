**EXPERIMENTAL**
Please note that this repository currently contains documents and artifacts which are "work in progress" and as such should be used with caution under the guidance of the Mojaloop Foundation.
At such time as the content of this repository has been sufficiently tested this notice will be removed.


This repository contains code, documentation and other artefacts that support the deployment of Mojaloop software in production quality on-premises scenarios.

## On-Premises Deployment Recommendations

The Mojaloop Foundation recommends:
1. That on-premises deployments be one or more (e.g. for environment separation) single-purpose cluster(s) of commodity bare-metal server nodes.
2. That the bare-metal server nodes should run [Kairos](kairos.io) immutable Kubernetes on Linux operating systems for hosting Mojaloop software containers.


## Cluster Provisioning
Documentation covering cluster provisioning can be found [here](docs/Cluster%20Provisining.md).

## Software Deployment
Documentation covering Mojaloop software deployment can be found [here](docs/Software%20Deployment%20Recommendations.md).
