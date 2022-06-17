Tekton
```
Task: a reusable, loosely coupled number of steps that perform a specific task (e.g. building a container image)
Pipeline: the definition of the pipeline and the Tasks that it should perform
TaskRun: the execution and result of running an instance of a task
PipelineRun: the execution and result of running an instance of a pipeline, which includes a number of TaskRuns
```
```
Task - A Task defines a series of steps that run in a desired order and complete a set amount of build work. Every Task runs as a Pod on your Kubernetes cluster with each step as its own container.

Pipeline - A Pipeline defines an ordered series of Tasks that you want to execute along with the corresponding inputs and outputs for each Task. In fact, tasks should do one single thing so you can reuse them across pipelines or even within a single pipeline.
```

```
oc get pods -n openshift-console | grep console
oc get routes console -n openshift-console
```

- subscription.yaml
```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

```
oc apply -f subscription.yaml
oc new-project pipelines-tutorial
```

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: say-hello
      image: registry.access.redhat.com/ubi8/ubi
      command:
        - /bin/bash
      args: ['-c', 'echo Hello World']
```
```
oc apply -f https://raw.githubusercontent.com/openshift-labs/learn-katacoda/master/middleware/pipelines/assets/tasks/hello.yaml
```
```
tkn task start --showlog hello
```
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: apply-manifests
spec:
  workspaces:
  - name: source
  params:
    - name: manifest_dir
      description: The directory in source that contains yaml manifests
      type: string
      default: "k8s"
  steps:
    - name: apply
      image: quay.io/openshift/origin-cli:latest
      workingDir: /workspace/source
      command: ["/bin/bash", "-c"]
      args:
        - |-
          echo Applying manifests in $(inputs.params.manifest_dir) directory
          oc apply -f $(inputs.params.manifest_dir)
          echo -----------------------------------
```
```
oc create -f https://raw.githubusercontent.com/openshift-labs/learn-katacoda/master/middleware/pipelines/assets/tasks/apply_manifest_task.yaml
```
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: update-deployment
spec:
  params:
    - name: deployment
      description: The name of the deployment patch the image
      type: string
    - name: IMAGE
      description: Location of image to be patched with
      type: string
  steps:
    - name: patch
      image: quay.io/openshift/origin-cli:latest
      command: ["/bin/bash", "-c"]
      args:
        - |-
          oc patch deployment $(inputs.params.deployment) --patch='{"spec":{"template":{"spec":{
            "containers":[{
              "name": "$(inputs.params.deployment)",
              "image":"$(inputs.params.IMAGE)"
            }]
          }}}}'
```
```
oc create -f https://raw.githubusercontent.com/openshift-labs/learn-katacoda/master/middleware/pipelines/assets/tasks/update_deployment_task.yaml
```
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: source-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```
oc create -f https://raw.githubusercontent.com/openshift-labs/learn-katacoda/master/middleware/pipelines/assets/resources/persistent_volume_claim.yaml
```

- Listing the task created
```
tkn task ls
```

- Pipeline
```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  workspaces:
  - name: shared-workspace
  params:
  - name: deployment-name
    type: string
    description: name of the deployment to be patched
  - name: git-url
    type: string
    description: url of the git repo for the code of deployment
  - name: git-revision
    type: string
    description: revision to be used from repo of the code for deployment
    default: "master"
  - name: IMAGE
    type: string
    description: image to be build from the code
  tasks:
  - name: fetch-repository
    taskRef:
      name: git-clone
      kind: ClusterTask
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.git-url)
    - name: subdirectory
      value: ""
    - name: deleteExisting
      value: "true"
    - name: revision
      value: $(params.git-revision)
  - name: build-image
    taskRef:
      name: buildah
      kind: ClusterTask
    params:
    - name: TLSVERIFY
      value: "false"
    - name: IMAGE
      value: $(params.IMAGE)
    workspaces:
    - name: source
      workspace: shared-workspace
    runAfter:
    - fetch-repository
  - name: apply-manifests
    taskRef:
      name: apply-manifests
    workspaces:
    - name: source
      workspace: shared-workspace
    runAfter:
    - build-image
  - name: update-deployment
    taskRef:
      name: update-deployment
    workspaces:
    - name: source
      workspace: shared-workspace
    params:
    - name: deployment
      value: $(params.deployment-name)
    - name: IMAGE
      value: $(params.IMAGE)
    runAfter:
    - apply-manifests
```
```
oc create -f https://raw.githubusercontent.com/openshift-labs/learn-katacoda/master/middleware/pipelines/assets/pipeline/pipeline.yaml
```

Trigger the pipeline via CLI
```
tkn pipeline start build-and-deploy -w name=shared-workspace,claimName=source-pvc -p deployment-name=pipelines-vote-api -p git-url=https://github.com/openshift/pipelines-vote-api.git -p IMAGE=image-registry.openshift-image-registry.svc:5000/pipelines-tutorial/vote-api -p git-revision=master --showlog
```

```
tkn pipeline start build-and-deploy -w name=shared-workspace,claimName=source-pvc -p deployment-name=pipelines-vote-ui -p git-url=https://github.com/openshift/pipelines-vote-ui.git -p IMAGE=image-registry.openshift-image-registry.svc:5000/pipelines-tutorial/vote-ui -p git-revision=master --showlog
```

GitOps

```
GitOps is a set of practices that leverages Git workflows to manage infrastructure and application configurations. By using Git repositories as the source of truth, it allows the DevOps team to store the entire state of the cluster configuration in Git so that the trail of changes are visible and auditable.

GitOps simplifies the propagation of infrastructure and application configuration changes across multiple clusters by defining your infrastructure and applications definitions as “code”.

Ensure that the clusters have similar states for configuration, monitoring, or storage.
Recover or recreate clusters from a known state.
Create clusters with a known state.
Apply or revert configuration changes to multiple clusters.
Associate templated configuration with different environments.
```


```
Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

It follows the GitOps pattern of using Git repositories as the source of truth for defining the desired application state.

It automates the deployment of the desired application states in the specified target environments. Application deployments can track updates to branches, tags, or pinned to a specific version of manifests at a Git commit.

```

```
// on your local machine
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

// Deploy the operators
oc get operators
oc get pods -n openshift-gitops

// connecting to argo cd server
oc get routes -n openshift-gitops | grep openshift-gitops-server | awk '{print $2}'

// getting the password for argocd
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-

// login to argo cd using CLI
argocd login $ARGOCD_SERVER_URL

// list available servers
argocd cluster list
source <(argocd completion bash)

// argocd confifuration
ls ~/.argocd/config
cat  ~/.argocd/config
```

- App
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgd-app
  namespace: openshift-gitops
spec:
  destination:
    namespace: bgd
    server: https://kubernetes.default.svc
  project: default
  source:
    path: apps/bgd/overlays/bgd
    repoURL: https://github.com/redhat-developer-demos/openshift-gitops-examples
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
```
```
oc apply -f https://raw.githubusercontent.com/redhat-developer-demos/openshift-gitops-examples/main/components/applications/bgd-app.yaml
oc get pods,svc,route -n bgd
oc rollout status deploy/bgd -n bgd
oc get route -n bgd
oc -n bgd patch deploy/bgd --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/env/0/value", "value":"green"}]'
oc rollout status deploy/bgd -n bgd
argocd app sync bgd-app
oc patch application/bgd-app -n openshift-gitops --type=merge -p='{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

kustomize

```
kustomize version --short
cd /opt
git clone https://github.com/redhat-developer-demos/openshift-gitops-examples.git
cd openshift-gitops-examples
cd components/kustomize-build/
kustomize build .
kustomize edit set image quay.io/redhatworkshops/welcome-php:ffcd15
kustomize build . | oc apply -f -
kubectl kustomize --help
oc new-project kustomize-test
kubectl apply -k ./
kubectl get pods -n kustomize-test
kubectl get deployment welcome-php -o jsonpath='{.metadata.labels}' | jq -r
kubectl get deploy welcome-php  -o jsonpath='{.spec.template.spec.containers[].image}{"\n"}'

oc apply -f components/applications/bgd-app.yaml
oc apply -f components/applications/bgdk-app.yaml
oc get 
```
