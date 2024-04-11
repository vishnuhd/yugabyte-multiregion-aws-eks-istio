# Multi-region Yubagyte deployment on AWS EKS with Istio

This repo shows how to install Yugabyte across multiple regions and clusters on AWS EKS with Istio.

## Pre-requisites

- AWS account with atleast three regions enabled
- AWS user with access to create VPC and EKS using `eksctl`
- `eksctl` installed
- `aws cli` configured and installed
- `kubectl` installed

## Architecture

![Screenshot 2024-04-11 at 6 26 49 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/87a0b24c-5fea-453c-9057-1de64dbf0c44)


## Deploy AWS EKS clusters

- Deploy EKS clusters in two different regions :

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Creating EKS cluster in ${region}...\n"
    eksctl create cluster -f ${region}/cluster-config.yaml
    echo -e "-------------\n"
  done
}
```

- Rename the kube contexts for simiplicity of the demo:

```sh 
kubectl config rename-context 'yb-mumbai.ap-south-1.eksctl.io' mumbai
kubectl config rename-context 'yb-singapore.ap-southeast-1.eksctl.io' singapore
kubectl config rename-context 'yb-hyderabad.ap-south-2.eksctl.io' hyderabad
```

NOTE: By default for EKS does not provide EBS permissions, follow [this](https://repost.aws/knowledge-center/eks-persistent-storage) article to enable EKS PVC dynamic provisioning.

## Setup Istio

- Download istio

```sh 
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.21.0
export PATH=$PWD/bin:$PATH
```

## Plug in CA Certificates

- Create directories :

```sh 
mkdir -p istio-1.21.0/certs
```

- Generate the root certificate and key:

```sh 
cd istio-1.21.0/certs
make -f ../tools/certs/Makefile.selfsigned.mk root-ca

```

- For each cluster, generate an intermediate certificate and key for the Istio CA:

```sh 
cd istio-1.21.0/certs

{
  for region in mumbai singapore hyderabad; do
    echo -e "Generating certs for cluster - ${region}...\n"
    make -f ../tools/certs/Makefile.selfsigned.mk yb-${region}-cacerts
    echo -e "-------------\n"
  done
}
```

- In each cluster, create a secret cacerts including all the input files `ca-cert.pem`, `ca-key.pem`, `root-cert.pem` and `cert-chain.pem` :

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Creating namespace and secret for cluster - ${region}...\n"
    
    kubectl --context ${region} create namespace istio-system
    kubectl --context ${region} create secret generic cacerts -n istio-system \
          --from-file=istio-1.21.0/certs/yb-${region}/ca-cert.pem \
          --from-file=istio-1.21.0/certs/yb-${region}/ca-key.pem \
          --from-file=istio-1.21.0/certs/yb-${region}/root-cert.pem \
          --from-file=istio-1.21.0/certs/yb-${region}/cert-chain.pem

    echo -e "-------------\n"
  done
}
```

## Install Istio

- Install istio using `istioctl`:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Installing istio for cluster - ${region}...\n"
    
    istioctl install --context ${region} -f ./${region}/istio.yaml

    echo -e "-------------\n"
  done
}
```

- Install the east-west gateway:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Installing the east-west gateway for cluster - ${region}...\n"
    
    ./istio-1.21.0/samples/multicluster/gen-eastwest-gateway.sh \
        --mesh mesh1 --cluster yb-${region} --network network-${region} | \
        istioctl --context ${region} install -y -f -

    echo -e "-------------\n"
  done
}
```

- Expose the internal `.local` k8s services:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Exposing the services for cluster - ${region}...\n"
    
    kubectl --context ${region} apply -n istio-system -f \
        ./istio-1.21.0/samples/multicluster/expose-services.yaml

    echo -e "-------------\n"
  done
}
```

- Enable Endpoint Discovery:

```sh 
{
  for region1 in mumbai singapore hyderabad; do
    for region2 in mumbai singapore hyderabad; do
      if [[ "${region1}" == "${region2}" ]]; then continue; fi
      echo -e "Create remote secret of ${region1} in ${region2}...\n"
      
      istioctl create-remote-secret \
        --context ${region1} \
        --name=yb-${region1} | \
        kubectl apply -f - --context ${region2}

      echo -e "-------------\n"
    done
  done
}
```

## Install Yugabyte

- Prepare helm repo:

```sh 
helm repo add yugabytedb https://charts.yugabyte.com
helm repo update
```

- Create namespaces and inject istio:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Creating namespace for cluster - ${region}...\n"
    
    kubectl --context ${region} create namespace yb-demo
    kubectl label --context ${region} namespace yb-demo istio-injection=enabled

    echo -e "-------------\n"
  done
}
```

- Install YugabyteDB:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Installing YugabyteDB in cluster - ${region}...\n"
    
    helm upgrade --install ${region} yugabytedb/yugabyte \
        --version 2.19.3 \
        --namespace yb-demo \
        -f ${region}/overrides.yaml \
        --kube-context ${region} --wait

    echo -e "-------------\n"
  done
}
```

This will deploy 1 YB master in each cluster

- Create additional k8s services for istio multi-cluster setup:

```sh 
{
  for region1 in mumbai singapore hyderabad; do
    for region2 in mumbai singapore hyderabad; do
      if [[ "${region1}" == "${region2}" ]]; then continue; fi
      echo -e "Creating services of ${region2} in ${region1}...\n"
      
      kubectl --context ${region1} apply -f ${region1}/services-${region2}.yaml -n yb-demo

      echo -e "-------------\n"
    done
  done
}
```

- Check the YugabyteDB pods and services:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Checking YugabyteDB pods and svcs for cluster - ${region}...\n"
    
    kubectl --context ${region} get pods,svc -A

    echo -e "-------------\n"
  done
}
```

- Configure global data distribution:

```sh 
kubectl --context mumbai exec -n yb-demo mumbai-yugabyte-yb-master-0 -- bash \
-c "/home/yugabyte/master/bin/yb-admin --master_addresses mumbai-yugabyte-yb-master-0.yb-demo.svc.cluster.local,hyderabad-yugabyte-yb-master-0.yb-demo.svc.cluster.local,singapore-yugabyte-yb-master-0.yb-demo.svc.cluster.local modify_placement_info aws.ap-south-1.ap-south-1a,aws.ap-south-2.ap-south-2a,aws.ap-southeast-1.ap-southeast-1a 3"
```

- Now that the YugabyteDB installation is successful, open the master UI in browser:

![Screenshot 2024-04-11 at 6 05 49 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/c6a38c8e-04cc-4327-9155-79657ee55db8)

- Tablet servers:

![Screenshot 2024-04-11 at 6 06 24 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/f8af3548-a8cb-468e-ab43-fd24d4aaeddd)

- User tables:

![Screenshot 2024-04-11 at 6 07 51 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/a59539ac-d2cd-47d4-9c3b-3ba5c4b385cf)

- Run YB sample app:

```sh 
kubectl run yb-sample-apps \
    -it --rm \
    --image yugabytedb/yb-sample-apps \
    --namespace yb-demo \
    --context singapore \
    --command -- sh

java -jar yb-sample-apps.jar java-client-sql \
    --workload SqlInserts \
    --nodes yb-tserver-common.yb-demo.svc.cluster.local:5433 \
    --num_threads_write 1 \
    --num_threads_read 2
```

## Disaster recovery

- Current leaders for master and tablet servers:

![Screenshot 2024-04-11 at 6 27 27 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/ff40d424-fb1c-442f-a8f7-12607bf25dd2)
![Screenshot 2024-04-11 at 6 30 29 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/593ac280-94b0-46da-bfdd-b382036c16ee)

- Once any region (`hyderabad`) region goes down, a new master and a new tablet server are elected as leaders respectively:

![Screenshot 2024-04-11 at 6 31 54 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/54610491-0ed8-466b-b081-0bd32a6f9da9)
![Screenshot 2024-04-11 at 6 32 03 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/98fe8605-6d73-4cf4-8fdf-dc2bb6c18f35)

- Yugabyte keeps track of under-replicated tables:

![Screenshot 2024-04-11 at 6 33 07 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/058e5aba-7625-4e54-8a16-51cf565497ba)

- Once the region is restored, the transactions are replicated to the region:

![Screenshot 2024-04-11 at 6 34 23 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/8b349be7-7bc1-444a-8d69-54b81a7f7f83)
![Screenshot 2024-04-11 at 6 34 29 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/6e8463b8-3779-4a46-acef-759f930c1f51)
![Screenshot 2024-04-11 at 6 34 35 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/0027b324-692b-4fd2-b9f0-fd9785d52240)


## Setup Kiali for observability

- Install Kiali and Prometheus:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Checking Kiali and Prometheus for cluster - ${region}...\n"
    
    kubectl apply -f istio-1.21.0/samples/addons/prometheus.yaml --context ${region}
    kubectl apply -f istio-1.21.0/samples/addons/kiali.yaml --context ${region}

    echo -e "-------------\n"
  done
}
```

- Open Kiali dashboard:

```sh 
istioctl dashboard kiali --context singapore
```

![Screenshot 2024-04-11 at 6 26 49 PM](https://github.com/vishnuhd/yugabyte-multiregion-aws-eks-istio/assets/35323586/06c5e4ac-de5c-43eb-be2a-071a7bee1162)

## Cleaning up the resources

- Uninstall YugabyteDB:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Un-installing YugabyteDB in cluster - ${region}...\n"
    
    helm uninstall ${region} --namespace yb-demo --kube-context ${region}
    kubectl delete pvc --namespace yb-demo \
      --selector component=yugabytedb,release=${region} \
      --context ${region}

    echo -e "-------------\n"
  done
}
```

- Delete additional YB services:

```sh 
{
  for region1 in mumbai singapore hyderabad; do
    for region2 in mumbai singapore hyderabad; do
      if [[ "${region1}" == "${region2}" ]]; then continue; fi
      echo -e "Deleting services of ${region2} in ${region1}...\n"
      
      kubectl --context ${region1} delete -f ${region1}/services-${region2}.yaml -n yb-demo

      echo -e "-------------\n"
    done
  done
}
```

- Un-install Kiali and Prometheus:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Checking Kiali and Prometheus for cluster - ${region}...\n"
    
    kubectl delete -f istio-1.21.0/samples/addons/prometheus.yaml --context ${region}
    kubectl delete -f istio-1.21.0/samples/addons/kiali.yaml --context ${region}

    echo -e "-------------\n"
  done
}
```

- Uninstall istio:

```sh 
{
  for region in mumbai singapore hyderabad; do
    echo -e "Un-installing Istio in cluster - ${region}...\n"
    
    istioctl uninstall --purge -y --context ${region}

    echo -e "-------------\n"
  done
}
```

- Delete EKS clusters:

```sh 
eksctl delete cluster yb-mumbai --region ap-south-1
eksctl delete cluster yb-singapore --region ap-southeast-1
eksctl delete cluster yb-hyderabad --region ap-south-2
```
