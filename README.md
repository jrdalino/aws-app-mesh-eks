# aws-appmesh-eks

## Prerequisites
- k8s >= 1.12
- kubectl
- jq

## Step 1: Setup AppMesh
- Create a Mesh and Virtual Service
- Create a Virutal Node
- Create a Virtual Router and Route
- Create additional resources
- Update Services

## Step 2: Install the Controller and Custom Resources
- Attach AWSAppMeshFullAccess Policy to Kubernetes Worker Nodes
- Create the Kubernetes custom resources and launch the controller,
```
$ curl https://raw.githubusercontent.com/aws/aws-app-mesh-controller-for-k8s/master/deploy/all.yaml | kubectl apply -f -
```
- Confirm that the controller is running with the following command.
```
$ kubectl rollout status deployment app-mesh-controller -n appmesh-system
```
```
deployment "app-mesh-controller" successfully rolled out
```
- Confirm that the Kubernetes custom resources for App Mesh were created with the following command.
```
kubectl get crd
```
```
NAME                               CREATED AT
meshes.appmesh.k8s.aws             2019-05-08T14:17:26Z
virtualnodes.appmesh.k8s.aws       2019-05-08T14:17:26Z
virtualservices.appmesh.k8s.aws    2019-05-08T14:17:26Z
```

## Step 3: Install Sidecar Injector
- Export name and region of mesh
```
$ export MESH_NAME=my-mesh
$ export MESH_REGION=region
```
- Download and execute the sidecar injector installation script with the following command.
```
$ curl https://raw.githubusercontent.com/aws/aws-app-mesh-inject/master/scripts/install.sh | bash
```
```
deployment.apps/appmesh-inject created
mutatingwebhookconfiguration.admissionregistration.k8s.io/appmesh-inject created
waiting for aws-app-mesh-inject to start
Waiting for deployment "appmesh-inject" rollout to finish: 0 of 1 updated replicas are available...
deployment "appmesh-inject" successfully rolled out
Mesh name has been set up
The injector is ready
```

## Step 4: Configure App Mesh
- Create Kubernetes Custom Resources
- Create a Mesh
```
apiVersion: appmesh.k8s.aws/v1beta1
kind: Mesh
metadata:
  name: my-mesh
```
- Create a Virtual Service
```
apiVersion: appmesh.k8s.aws/v1beta1
kind: VirtualService
metadata:
  name: my-svc-a
  namespace: my-namespace
spec:
  meshName: my-mesh
  routes:
    - name: route-to-svc-a
      http:
        match:
          prefix: /
        action:
          weightedTargets:
            - virtualNodeName: my-app-a
              weight: 1
```
- Create a Virtual Node
```
apiVersion: appmesh.k8s.aws/v1beta1
kind: VirtualNode
metadata:
  name: my-app-a
  namespace: my-namespace
spec:
  meshName: my-mesh
  listeners:
    - portMapping:
        port: 9000
        protocol: http
  serviceDiscovery:
    dns:
      hostName: my-app-a.my-namespace.svc.cluster.local
  backends:
    - virtualService:
        virtualServiceName: my-svc-a
```
- Enable Sidecar Injection for a Namespace
```
$ kubectl label namespace my-namespace appmesh.k8s.aws/sidecarInjectorWebhook=enabled
```
- Override Sidecar Injector Default Behavior
```
apiVersion: appmesh.k8s.aws/v1beta1
kind: Deployment
spec:
    metadata:
      annotations:
        appmesh.k8s.aws/mesh: my-mesh2
        appmesh.k8s.aws/ports: "8079,8080"
        appmesh.k8s.aws/egressIgnoredPorts: "3306"
        appmesh.k8s.aws/virtualNode: my-app
        appmesh.k8s.aws/sidecarInjectorWebhook: disabled
```

## Step 5: Deploy a Mesh Connected Service
- To deploy the color mesh sample application, download the following file and apply it to your Kubernetes cluster with the following command.
```
$ curl https://raw.githubusercontent.com/aws/aws-app-mesh-controller-for-k8s/master/examples/color.yaml | kubectl apply -f -
```
- View the resources deployed by the sample application with the following command.
```
$ kubectl -n appmesh-demo get all
```

## Step 6: Run an Application
- In a terminal, use the following command to create a container in the appmesh-demo namespace that has curl installed and open a shell to it. In later steps, this terminal is referred to as Terminal A.
```
$ kubectl run -n appmesh-demo -it curler --image=tutum/curl /bin/bash
```
- From Terminal A, run the following command to curl the color gateway in the color mesh application 100 times. The gateway routes traffic to separate virtual nodes that return either white, black, or blue as a response.
```
for i in {1..100}; do curl colorgateway:9080/color; echo; done
```
```
{"color":"blue", "stats": {"black":0.36,"blue":0.32,"white":0.32}}
```

## Step 7: Change Configuration
- In a separate terminal from Terminal A, edit the colorteller.appmesh-demo virtual service with the following command.
```
kubectl edit VirtualService colorteller.appmesh-demo -n appmesh-demo
```
```
spec:
  meshName: color-mesh
  routes:
  - http:
      action:
        weightedTargets:
        - virtualNodeName: colorteller.appmesh-demo
          weight: 0
        - virtualNodeName: colorteller-blue
          weight: 0
        - virtualNodeName: colorteller-black.appmesh-demo
          weight: 1
```
- In Terminal A, run curl again with the following command.
```
for i in {1..100}; do curl colorgateway:9080/color; echo; done
```
```
{"color":"black", "stats": {"black":0.64,"blue":0.18,"white":0.19}}
```

## Step 8: Remove Application and Integration Components (Optional)
- Application
```
$ kubectl delete namespace appmesh-demo
$ kubectl delete mesh color-mesh
```
- Integration
```
$ kubectl delete crd meshes.appmesh.k8s.aws
$ kubectl delete crd virtualnodes.appmesh.k8s.aws
$ kubectl delete crd virtualservices.appmesh.k8s.aws
$ kubectl delete namespace appmesh-system
$ kubectl delete namespace appmesh-inject
```

## References:
- https://docs.aws.amazon.com/app-mesh/latest/userguide/appmesh-getting-started.html
- https://docs.aws.amazon.com/app-mesh/latest/userguide/mesh-k8s-integration.html
- https://docs.aws.amazon.com/app-mesh/latest/userguide/deploy-mesh-connected-service.html
