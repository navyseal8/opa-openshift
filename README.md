# OPA gatekeeper installation and Rego rules demo

# Modify Security Context Constraint (SCC)
- From K8S v1.21 onwards, PSP is deprecated. OpenShift retain the use of SCC, which is similar to PSP but more feature rich. 
- The default gatekeeper installation yaml was not suitable to be deploy on a OpenShift cluster (and it's managed cluster like ROSA/ARO)


## SCC for gatekeeper-admin

Replace PSP codes with SCC. Non-root, least privilege

```
$ oc describe scc gatekeeper-admin 
Name:                                           gatekeeper-admin
Priority:                                       <none>
Access:
  Users:                                        gatekeeper-admin
  Groups:                                       <none>
Settings:
  Allow Privileged:                             false
  Allow Privilege Escalation:                   false
  Default Add Capabilities:                     <none>
  Required Drop Capabilities:                   ALL
  Allowed Capabilities:                         <none>
  Allowed Seccomp Profiles:                     runtime/default
  Allowed Volume Types:                         configMap,downwardAPI,emptyDir,projected,secret
  Allowed Flexvolumes:                          <all>
  Allowed Unsafe Sysctls:                       <none>
  Forbidden Sysctls:                            <none>
  Allow Host Network:                           false
  Allow Host Ports:                             false
  Allow Host PID:                               false
  Allow Host IPC:                               false
  Read Only Root Filesystem:                    false
  Run As User Strategy: MustRunAsNonRoot
    UID:                                        <none>
    UID Range Min:                              <none>
    UID Range Max:                              <none>
  SELinux Context Strategy: RunAsAny
    User:                                       <none>
    Role:                                       <none>
    Type:                                       <none>
    Level:                                      <none>
  FSGroup Strategy: MustRunAs
    Ranges:                                     1-65535
  Supplemental Groups Strategy: MustRunAs
    Ranges:                                     1-65535
```

## Namespace 

Setting uid-range to allow for random UID to be within range

```
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  labels:
    admission.gatekeeper.sh/ignore: no-self-managing
    control-plane: controller-manager
    gatekeeper.sh/system: "yes"
  annotations:
    openshift.io/sa.scc.mcs: s0:c26,c5
    openshift.io/sa.scc.supplemental-groups: 999/10000
    openshift.io/sa.scc.uid-range: 1000/10000
  name: gatekeeper-system
```

## Installing Gatekeeper

```
$ oc apply -f gatekeeper.yaml 
project.project.openshift.io/gatekeeper-system created
resourcequota/gatekeeper-critical-pods created
customresourcedefinition.apiextensions.k8s.io/assign.mutations.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/assignmetadata.mutations.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/configs.config.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constraintpodstatuses.status.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constrainttemplatepodstatuses.status.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constrainttemplates.templates.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/modifyset.mutations.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/mutatorpodstatuses.status.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/providers.externaldata.gatekeeper.sh created
serviceaccount/gatekeeper-admin created
securitycontextconstraints.security.openshift.io/gatekeeper-admin created
role.rbac.authorization.k8s.io/gatekeeper-manager-role created
clusterrole.rbac.authorization.k8s.io/gatekeeper-manager-role created
rolebinding.rbac.authorization.k8s.io/gatekeeper-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/gatekeeper-manager-rolebinding created
secret/gatekeeper-webhook-server-cert created
service/gatekeeper-webhook-service created
deployment.apps/gatekeeper-audit created
deployment.apps/gatekeeper-controller-manager created
poddisruptionbudget.policy/gatekeeper-controller-manager created
mutatingwebhookconfiguration.admissionregistration.k8s.io/gatekeeper-mutating-webhook-configuration created
validatingwebhookconfiguration.admissionregistration.k8s.io/gatekeeper-validating-webhook-configuration created
```

Verify all deployment are successful

```
oc get all -n gatekeeper-system
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/gatekeeper-audit-54fdfd965-nhm5x                 1/1     Running   0          32s
pod/gatekeeper-controller-manager-798fb78f97-gd2rf   1/1     Running   0          32s
pod/gatekeeper-controller-manager-798fb78f97-k452g   1/1     Running   0          32s
pod/gatekeeper-controller-manager-798fb78f97-sfmf5   1/1     Running   0          32s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/gatekeeper-webhook-service   ClusterIP   172.30.140.74   <none>        443/TCP   32s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gatekeeper-audit                1/1     1            1           32s
deployment.apps/gatekeeper-controller-manager   3/3     3            3           32s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/gatekeeper-audit-54fdfd965                 1         1         1       32s
replicaset.apps/gatekeeper-controller-manager-798fb78f97   3         3         3       32s
```

## Testing a Rego rule

Create the first namespace constraint. w/o a label of "gatekeeper", any namespace creation will fail.
```
$ oc apply -f constraint.yaml 
constrainttemplate.templates.gatekeeper.sh/k8srequiredlabels created

$ oc get constrainttemplate k8srequiredlabels
NAME                AGE
k8srequiredlabels   85s
```

Create your rule

```
$ oc apply -f rule1.yaml 
cat rk8srequiredlabels.constraints.gatekeeper.sh/ns-must-have-gk created

$ oc new-project test
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: [ns-must-have-gk] you must provide labels: {"gatekeeper"}

$ oc apply -f good-label.yaml 
namespace/good-ns unchanged

$ oc describe project good-ns
Name:                   good-ns
Created:                15 hours ago
Labels:                 gatekeeper=true
```

