# Nebula Operator Kind, oneliner installer for Nebula K8S Operator Playground 


> Nebula-Kind, an one-liner command to try K8S Operator based Nebula Graph Cluster on your machine, with the help of KIND (K8S in Docker)

<!--more-->

## Nebula-Operator-Kind

As a Cloud Native Distributed Database, Nebula Graph comes with an open-source [K8S Operator](https://github.com/vesoft-inc/nebula-operator) to enable boostrap and maintain Nebula Graph Cluster from a K8S CRD.

Normally it takes you some time to setup all the dependencies and control plane resources of the Nebula Operator. If you are as lazy as I am, this Nebula-Operator-Kind is made for you to quick start and play with Nebula Graph in [KIND](https://kind.sigs.k8s.io/).

Nebula-Operator-Kind is the one-liner for setup everything for you including:
- Docker
- K8S(KIND)
- PV Provider
- Nebula-Operator
- Nebula-Console
- nodePort for accessing the Cluster
- Kubectl for playing with KIND and Nebula Operator

## How To Use
Install Nebula-Operator-Kind:
```bash
curl -sL nebula-kind.siwei.io/install.sh | bash
```
You will see this after it's done
![install_success](./install_success.webp)

You can connect to the cluster via `~/.nebula-kind/bin/console` as below:
```bash
~/.nebula-kind/bin/console -u user -p password --address=127.0.0.1 --port=30000
```

## More

It's in GitHub with more information you may be intrested in ;-), please try and feedback there~
https://github.com/wey-gu/nebula-operator-kind


> Banner Picture Credit: [Maik Hankemann](https://unsplash.com/photos/a4Gz2DD4dX0) 
