# 提交 Kubernetes 資源

從 Notebook 提交 Kubernetes 資源。

## Notebook Pod ServiceAccount

Kubeflow 將 `default-editor` 的 Kubernetes ServiceAccount 分配給筆記本 Pod。 在 Kubernetes 中 `default-editor` 這個 ServiceAccount 會跟 `kubeflow-edit` 的 ClusterRole 綁定，它對許多 Kubernetes 資源具有命名空間範圍內的權限。

您可以使用以下命令獲取 `ClusterRole/kubeflow-edit` 的 RBAC 的完整列表：

```bash
kubectl describe clusterrole kubeflow-edit
```

結果如下:

```
Name:         kubeflow-edit
Labels:       rbac.authorization.kubeflow.org/aggregate-to-kubeflow-admin=true
Annotations:  <none>
PolicyRule:
  Resources                                         Non-Resource URLs  Resource Names  Verbs
  ---------                                         -----------------  --------------  -----
  cronworkflows.argoproj.io/finalizers              []                 []              [*]
  cronworkflows.argoproj.io                         []                 []              [*]
  workfloweventbindings.argoproj.io                 []                 []              [*]
  workflows.argoproj.io/finalizers                  []                 []              [*]
  workflows.argoproj.io                             []                 []              [*]
  workflowtemplates.argoproj.io                     []                 []              [*]
  scheduledworkflows.kubeflow.org                   []                 []              [*]
  runs.pipelines.kubeflow.org                       []                 []              [archive create delete retry terminate unarchive reportMetrics readArtifact get list]
  experiments.pipelines.kubeflow.org                []                 []              [archive create delete unarchive get list]
  configmaps                                        []                 []              [create delete deletecollection patch update get list watch]
  endpoints                                         []                 []              [create delete deletecollection patch update get list watch]
  persistentvolumeclaims                            []                 []              [create delete deletecollection patch update get list watch]
  replicationcontrollers/scale                      []                 []              [create delete deletecollection patch update get list watch]
  replicationcontrollers                            []                 []              [create delete deletecollection patch update get list watch]
  services                                          []                 []              [create delete deletecollection patch update get list watch]
  daemonsets.apps                                   []                 []              [create delete deletecollection patch update get list watch]
  deployments.apps/scale                            []                 []              [create delete deletecollection patch update get list watch]
  deployments.apps                                  []                 []              [create delete deletecollection patch update get list watch]
  replicasets.apps/scale                            []                 []              [create delete deletecollection patch update get list watch]
  replicasets.apps                                  []                 []              [create delete deletecollection patch update get list watch]
  statefulsets.apps/scale                           []                 []              [create delete deletecollection patch update get list watch]
  statefulsets.apps                                 []                 []              [create delete deletecollection patch update get list watch]
  horizontalpodautoscalers.autoscaling              []                 []              [create delete deletecollection patch update get list watch]
  cronjobs.batch                                    []                 []              [create delete deletecollection patch update get list watch]
  jobs.batch                                        []                 []              [create delete deletecollection patch update get list watch]
  daemonsets.extensions                             []                 []              [create delete deletecollection patch update get list watch]
  deployments.extensions/scale                      []                 []              [create delete deletecollection patch update get list watch]
  deployments.extensions                            []                 []              [create delete deletecollection patch update get list watch]
  ingresses.extensions                              []                 []              [create delete deletecollection patch update get list watch]
  networkpolicies.extensions                        []                 []              [create delete deletecollection patch update get list watch]
  replicasets.extensions/scale                      []                 []              [create delete deletecollection patch update get list watch]
  replicasets.extensions                            []                 []              [create delete deletecollection patch update get list watch]
  replicationcontrollers.extensions/scale           []                 []              [create delete deletecollection patch update get list watch]
  ingresses.networking.k8s.io                       []                 []              [create delete deletecollection patch update get list watch]
  networkpolicies.networking.k8s.io                 []                 []              [create delete deletecollection patch update get list watch]
  poddisruptionbudgets.policy                       []                 []              [create delete deletecollection patch update get list watch]
  deployments.apps/rollback                         []                 []              [create delete deletecollection patch update]
  deployments.extensions/rollback                   []                 []              [create delete deletecollection patch update]
  jobs.pipelines.kubeflow.org                       []                 []              [create delete disable enable get list]
  mpijobs.kubeflow.org                              []                 []              [create delete get list patch update watch]
  mxjobs.kubeflow.org                               []                 []              [create delete get list patch update watch]
  paddlejobs.kubeflow.org                           []                 []              [create delete get list patch update watch]
  pytorchjobs.kubeflow.org                          []                 []              [create delete get list patch update watch]
  tfjobs.kubeflow.org                               []                 []              [create delete get list patch update watch]
  xgboostjobs.kubeflow.org                          []                 []              [create delete get list patch update watch]
  pipelines.pipelines.kubeflow.org/versions         []                 []              [create delete update get list]
  pipelines.pipelines.kubeflow.org                  []                 []              [create delete update get list]
  viewers.kubeflow.org                              []                 []              [create get delete]
  *.bindings.knative.dev                            []                 []              [create update patch delete get list watch]
  *.eventing.knative.dev                            []                 []              [create update patch delete get list watch]
  *.flows.knative.dev                               []                 []              [create update patch delete get list watch]
  *.messaging.knative.dev                           []                 []              [create update patch delete get list watch]
  *.serving.knative.dev                             []                 []              [create update patch delete get list watch]
  *.sources.knative.dev                             []                 []              [create update patch delete get list watch]
  visualizations.pipelines.kubeflow.org             []                 []              [create]
  notebooks.kubeflow.org                            []                 []              [get list create delete watch deletecollection patch update]
  notebooks.kubeflow.org/finalizers                 []                 []              [get list create delete]
  tensorboards.tensorboard.kubeflow.org/finalizers  []                 []              [get list create delete]
  tensorboards.tensorboard.kubeflow.org             []                 []              [get list create delete]
  pods/attach                                       []                 []              [get list watch create delete deletecollection patch update]
  pods/exec                                         []                 []              [get list watch create delete deletecollection patch update]
  pods/portforward                                  []                 []              [get list watch create delete deletecollection patch update]
  pods/proxy                                        []                 []              [get list watch create delete deletecollection patch update]
  secrets                                           []                 []              [get list watch create delete deletecollection patch update]
  services/proxy                                    []                 []              [get list watch create delete deletecollection patch update]
  *.istio.io                                        []                 []              [get list watch create delete deletecollection patch update]
  experiments.kubeflow.org                          []                 []              [get list watch create delete deletecollection patch update]
  notebooks.kubeflow.org/status                     []                 []              [get list watch create delete deletecollection patch update]
  suggestions.kubeflow.org                          []                 []              [get list watch create delete deletecollection patch update]
  trials.kubeflow.org                               []                 []              [get list watch create delete deletecollection patch update]
  *.networking.istio.io                             []                 []              [get list watch create delete deletecollection patch update]
  configurations.serving.knative.dev/status         []                 []              [get list watch create delete deletecollection patch update]
  configurations.serving.knative.dev                []                 []              [get list watch create delete deletecollection patch update]
  revisions.serving.knative.dev/status              []                 []              [get list watch create delete deletecollection patch update]
  revisions.serving.knative.dev                     []                 []              [get list watch create delete deletecollection patch update]
  routes.serving.knative.dev/status                 []                 []              [get list watch create delete deletecollection patch update]
  routes.serving.knative.dev                        []                 []              [get list watch create delete deletecollection patch update]
  services.serving.knative.dev/status               []                 []              [get list watch create delete deletecollection patch update]
  services.serving.knative.dev                      []                 []              [get list watch create delete deletecollection patch update]
  inferenceservices.serving.kserve.io               []                 []              [get list watch create delete deletecollection patch update]
  servingruntimes.serving.kserve.io                 []                 []              [get list watch create delete deletecollection patch update]
  poddefaults.kubeflow.org                          []                 []              [get list watch create delete]
  bindings                                          []                 []              [get list watch]
  events                                            []                 []              [get list watch]
  limitranges                                       []                 []              [get list watch]
  namespaces/status                                 []                 []              [get list watch]
  namespaces                                        []                 []              [get list watch]
  persistentvolumeclaims/status                     []                 []              [get list watch]
  pods/log                                          []                 []              [get list watch]
  pods/status                                       []                 []              [get list watch]
  replicationcontrollers/status                     []                 []              [get list watch]
  resourcequotas/status                             []                 []              [get list watch]
  resourcequotas                                    []                 []              [get list watch]
  services/status                                   []                 []              [get list watch]
  controllerrevisions.apps                          []                 []              [get list watch]
  daemonsets.apps/status                            []                 []              [get list watch]
  deployments.apps/status                           []                 []              [get list watch]
  replicasets.apps/status                           []                 []              [get list watch]
  statefulsets.apps/status                          []                 []              [get list watch]
  *.autoscaling.internal.knative.dev                []                 []              [get list watch]
  horizontalpodautoscalers.autoscaling/status       []                 []              [get list watch]
  cronjobs.batch/status                             []                 []              [get list watch]
  jobs.batch/status                                 []                 []              [get list watch]
  *.caching.internal.knative.dev                    []                 []              [get list watch]
  daemonsets.extensions/status                      []                 []              [get list watch]
  deployments.extensions/status                     []                 []              [get list watch]
  ingresses.extensions/status                       []                 []              [get list watch]
  replicasets.extensions/status                     []                 []              [get list watch]
  *.networking.internal.knative.dev                 []                 []              [get list watch]
  ingresses.networking.k8s.io/status                []                 []              [get list watch]
  poddisruptionbudgets.policy/status                []                 []              [get list watch]
  storageclasses.storage.k8s.io                     []                 []              [get list watch]
  mpijobs.kubeflow.org/status                       []                 []              [get]
  mxjobs.kubeflow.org/status                        []                 []              [get]
  paddlejobs.kubeflow.org/status                    []                 []              [get]
  pytorchjobs.kubeflow.org/status                   []                 []              [get]
  tfjobs.kubeflow.org/status                        []                 []              [get]
  xgboostjobs.kubeflow.org/status                   []                 []              [get]
  serviceaccounts                                   []                 []              [impersonate create delete deletecollection patch update get list watch]
  pods                                              []                 []              [list create delete deletecollection patch update get watch]
```

因為每個 Notebook Pod 都綁定了高權限的`default-editor` 這個 ServiceAccount，所以您可以在其中運行 `kubectl` 而無需提供額外的身份驗證。

例如，以下命令將創建 `test.yaml` 中定義的資源：

```bash
kubectl create -f "test.yaml" --namespace "MY_PROFILE_NAMESPACE"
```
