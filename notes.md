```
docker run -d --ulimit memlock=-1:-1 -it --rm=true --memory-swappiness=0 --name wanjadb -e POSTGRES_USER=wanja -e POSTGRES_PASSWORD=wanja -e POSTGRES_DB=wanjadb -p 5432:5432 postgres:10.5

quarkus ext add jib openshift


 oc new-app postgresql-persistent \
-p POSTGRESQL_USER=wanja \
-p POSTGRESQL_PASSWORD=wanja \
-p POSTGRESQL_DATABASE=wanjadb 

oc new-app --template=postgresql-ephemeral --param=POSTGRESQL_USER=wanja --param=POSTGRESQL_PASSWORD=wanja --param=POSTGRESQL_DATABASE=wanjadb

mvn package -Pnative -DskipTests
mvn package -Pnative -DskipTests -Dquarkus.native.container-build=true
mvn package -Pnative -DskipTests -Dquarkus.native.container-build=true -Dquarkus.container-image.push=true
mvn package -Pnative -DskipTests -Dquarkus.container-image.push=true

s2i
oc new-project book-dev
oc new-app --template=postgresql-ephemeral --param=POSTGRESQL_USER=wanja --param=POSTGRESQL_PASSWORD=wanja --param=POSTGRESQL_DATABASE=wanjadb
oc new-app java:openjdk-11-ubi8~https://github.com/mpaulgreen/gitops.git --context-dir=person-service --name=person-service
oc expose service/person-service

http http://person-service-book-dev.apps.kongcp.5xr9.p1.openshiftapps.com/person
http POST http://person-service-book-dev.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Jimi lastName=Hendrix salutation=Mr
http POST http://person-service-book-dev.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Joe lastName=Cocker salutation=Mr
http POST http://person-service-book-dev.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Carlos lastName=Santana salutation=Mr

raw kubernetes

oc new-project book-test
oc policy add-role-to-user system:image-puller system:serviceaccount:book-test:default --namespace=book-dev
oc new-app --template=postgresql-ephemeral --param=POSTGRESQL_USER=wanja --param=POSTGRESQL_PASSWORD=wanja --param=POSTGRESQL_DATABASE=wanjadb
oc apply -f raw-kubernetes/service.yaml
oc apply -f raw-kubernetes/deployment.yaml
oc apply -f raw-kubernetes/route.yaml


http http://person-service-book-test.apps.kongcp.5xr9.p1.openshiftapps.com/person
http POST http://person-service-book-test.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Jimi lastName=Hendrix salutation=Mr
http POST http://person-service-book-test.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Joe lastName=Cocker salutation=Mr
http POST http://person-service-book-test.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Carlos lastName=Santana salutation=Mr


templates

oc new-project book-template
oc policy add-role-to-user system:image-puller system:serviceaccount:book-template:default --namespace=book-dev
oc new-app --template=postgresql-ephemeral --param=POSTGRESQL_USER=wanja --param=POSTGRESQL_PASSWORD=wanja --param=POSTGRESQL_DATABASE=wanjadb
oc apply -f ocp-template/service-template.yaml
oc new-app service-template -p APPLICATION_NAME=simple-service
or
oc process service-template APPLICATION_NAME=process-service -o yaml | oc apply -f -

http http://simple-service-book-template.apps.kongcp.5xr9.p1.openshiftapps.com/person
http POST http://simple-service-book-template.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Jimi lastName=Hendrix salutation=Mr
http POST http://simple-service-book-template.apps.kongcp.5xr9.p1.openshiftapps.com//person firstName=Joe lastName=Cocker salutation=Mr
http POST http://simple-service-book-template.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Carlos lastName=Santana salutation=Mr

kustomize

oc new-project book-kustomize
oc policy add-role-to-user system:image-puller system:serviceaccount:book-kustomize:default --namespace=book-kustomize
oc new-app --template=postgresql-ephemeral --param=POSTGRESQL_USER=wanja --param=POSTGRESQL_PASSWORD=wanja --param=POSTGRESQL_DATABASE=wanjadb
kustomize build kustomize/overlays/dev
kustomize build kustomize/overlays/stage
kubectl apply -k kustomize/overlays/adv


// delete the dev deployment
oc apply -k kustomize/overlays/stage
kubectl apply -k kustomize/overlays/adv // do not work with oc (gotcha)
kubectl apply -k  kustomize/overlays/health/  // it overlays the deployment with health checks but will not work as the endpoints are missing

oc new-project book-health
kubectl apply -k kustomize/overlays/health



http http://stage-person-service-book-health.apps.kongcp.5xr9.p1.openshiftapps.com/person

tekton

mvn clean package -DskipTests
mvn package -DskipTests -Dquarkus.container-image.push=true
oc new-project book-tekton
tkn ct list // lists cluster tasks
tkn ct describe git-clone
tkn ct describe maven
tkn ct describe openshift-client
oc apply -f tekton/workspaces/
oc apply -f tekton/pipelines/tekton-pipeline.yaml
./tekton/pipeline.sh start -u mpaulgreen -p <<password> -t book-tekton
./tekton/pipeline.sh logs
```

http http://dev-person-service-book-tekton.apps.kongcp.5xr9.p1.openshiftapps.com/person
http POST http://dev-person-service-book-tekton.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Jimi lastName=Hendrix salutation=Mr
http POST http://dev-person-service-book-tekton.apps.kongcp.5xr9.p1.openshiftapps.com//person firstName=Joe lastName=Cocker salutation=Mr
http POST http://dev-person-service-book-tekton.apps.kongcp.5xr9.p1.openshiftapps.com/person firstName=Carlos lastName=Santana salutation=Mr


gitops

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

oc project openshift-gitops
oc apply -k gitops/argocd/

http http://dev-person-service-book-dev.apps.kongcp.5xr9.p1.openshiftapps.com/person
http http://stage-person-service-book-health.apps.kongcp.5xr9.p1.openshiftapps.com/person

oc delete -k gitops/argocd/

ci+gitops

oc new-project book-ci
oc apply -f secret.yaml