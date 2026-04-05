Mental Model

The k8s/ folder is not something Kubernetes reads automatically. It is just deployment source code.

Think of the Helm chart like this:

Chart.yaml = chart metadata
templates/*.yaml = blueprints
values.yaml = your input data
Helm = tool that combines templates + values
Kubernetes = cluster that runs the final rendered objects
So the flow is:

build and push image
point Helm values at that image
render the chart
apply it to the cluster
inspect pod health, logs, and mounted config
Your current GitLab pipeline in .gitlab-ci.yml only builds/tests/packages/pushes the image. It does not deploy the Helm chart yet. Deployment is still a separate step.

What To Do After You Populate values.yaml

For your first test, I recommend this sequence.

Make a separate test values file instead of editing the shared file forever.
Example:
cp k8s/helm/atlas-execution-processor/values.yaml \
  k8s/helm/atlas-execution-processor/values-uat1-amer-test.yaml
Put only one infra profile and one instance in that file.
That keeps the first test small and easy to debug.

Set the image details in that file:

image.repository = your GitLab registry image path
image.tag = the exact tag you want to deploy
Your pipeline pushes:

:$CI_COMMIT_SHA
:develop for develop builds
For learning/testing, I strongly recommend using the exact commit SHA tag, not develop, so you know exactly what image you are running.

Get cluster access working first.
You need:
kubectl
helm
kubeconfig from your platform team
the correct cluster context
Typical checks:

kubectl config get-contexts
kubectl config current-context
kubectl get ns
kubectl cluster-info
Pick a namespace and Helm release name.
Example:
namespace: aep-uat1-amer-test
release: aep-uat1-amer-test
Validate the chart before deploying.
Use:
helm lint ./k8s/helm/atlas-execution-processor \
  -f ./k8s/helm/atlas-execution-processor/values-uat1-amer-test.yaml
Render the chart to inspect what will be created.
Use:
helm template aep-uat1-amer-test ./k8s/helm/atlas-execution-processor \
  -n aep-uat1-amer-test \
  -f ./k8s/helm/atlas-execution-processor/values-uat1-amer-test.yaml
This does not deploy anything. It just shows you the final YAML.

Deploy it to the cluster.
Use:
helm upgrade --install aep-uat1-amer-test ./k8s/helm/atlas-execution-processor \
  -n aep-uat1-amer-test \
  --create-namespace \
  -f ./k8s/helm/atlas-execution-processor/values-uat1-amer-test.yaml
This is the main real deployment command.

What Helm Will Create

Based on your current chart:

configmaps.yaml creates:
one common ConfigMap
one infra ConfigMap
one flow ConfigMap
optional Kerberos ConfigMap
secrets.yaml creates optional infra Secret
deployments.yaml creates one Deployment per instance
services.yaml creates Services only if service.enabled=true
The Deployment mounts config into:

/config/common/application.yml
/config/infra/application.yml
/config/infra-secret/application.yml
/config/flow/application.yml
Spring reads those because the chart sets SPRING_CONFIG_LOCATION.

How To Verify The Deployment

After helm upgrade --install, run:

kubectl get deployments -n aep-uat1-amer-test
kubectl get pods -n aep-uat1-amer-test
kubectl get configmaps -n aep-uat1-amer-test
kubectl get secrets -n aep-uat1-amer-test
Then inspect one pod:

kubectl describe pod <pod-name> -n aep-uat1-amer-test
kubectl logs <pod-name> -n aep-uat1-amer-test
To check mounted config inside the container:

kubectl exec -it <pod-name> -n aep-uat1-amer-test -- ls -R /config
kubectl exec -it <pod-name> -n aep-uat1-amer-test -- cat /config/common/application.yml
kubectl exec -it <pod-name> -n aep-uat1-amer-test -- cat /config/infra/application.yml
kubectl exec -it <pod-name> -n aep-uat1-amer-test -- cat /config/flow/application.yml
That is one of the best learning steps because it makes the ConfigMap mounting model very clear.

How To Test The App Itself

Your chart currently has service.enabled: false, which is fine for initial testing.

So for first validation, use port-forward:

kubectl port-forward deployment/<deployment-name> 8080:8080 -n aep-uat1-amer-test
Then locally:

curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/health/readiness
curl http://localhost:8080/actuator/health/liveness
That lets you test the app without creating a Kubernetes Service.

Important Sanity Checks Before First Deploy

I would check these in your test values file before deploying:

image.repository points to the real GitLab registry path
image.tag points to a tag that actually exists
instances[0].infraProfile exactly matches an entry under infraProfiles
instance.instanceName is unique
if using POG_AMER_INTERNAL, queue-name is set
if using POG_AMER_INTERNAL_TOPIC, topic-name is set and queue-name is empty
serviceAccountName is correct if your cluster needs a specific service account
Kerberos and CSM bootstrap values are correct for your environment
Two specific observations from your current values.yaml:

secretConfig currently contains a DB password. If you are using normal CSM runtime, you probably should remove that and rely on aep.db.csm-secret-name.
solace.atlas.publisher.destination-type is currently topic; make sure that matches the actual AEP publisher behavior you want before testing.
How Changes Work Later

After the first deployment:

config-only change: edit values file, then run helm upgrade --install ...
new image: wait for pipeline to push image, update image.tag, then run helm upgrade --install ...
add new instance: add to instances, then run helm upgrade --install ...
Helm keeps release state, so use Helm again for updates. Do not mix normal Helm deployment with random kubectl apply changes on the same objects.

How To Clean Up After Testing

If you want to remove the test deployment:

helm uninstall aep-uat1-amer-test -n aep-uat1-amer-test
If you also want to remove the namespace:

kubectl delete namespace aep-uat1-amer-test
Best First Learning Loop

For your first pass, do only this:

one infra profile
one instance
one namespace
one exact image tag
helm lint
helm template
helm upgrade --install
kubectl get/describe/logs
kubectl exec into /config
kubectl port-forward and hit /actuator/health
That will teach you almost everything important about how this setup works.

If you want, I can next help you with a concrete first-test checklist against your current values.yaml, or give you the exact commands once you know the namespace, release name, and image tag you want to use.