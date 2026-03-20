# Kubernetes notes

custom crd , hpA , , crd 

## CLUSTER IN K8S

A **Kubernetes cluster** is a group of machines that work together to run containerized applications. Kubernetes manages them so your app stays up, scales, and heals itself.

### What a cluster is made of

### 1. Control Plane (the “brain” 🧠)

### 2. WORKER-NODE

# Revision files

[k8s.pdf](Kubernetes%20notes/k8s.pdf)

[k8s_troubleshooting_checklist.pdf](Kubernetes%20notes/k8s_troubleshooting_checklist.pdf)

## NodePort

### Full example (all ports visible)

```yaml
spec:
  type: NodePort
  selector:
    env: demo # Labels are env: demo
  ports:
  - nodePort: 30080  # Node port (external)  (first)
    port: 80        # Service port (cluster-ip) (second)
    targetPort: 8080 # Pod port (container)  (third-final)

```

---

### 🔄 Traffic flow (this is the key : if we use NodePort)

```
Internet
   ↓
NodeIP:30080
   ↓
Service port:80
   ↓
Pod port:8080
```

Each port has a **different responsibility**:

- `nodePort` → entry from outside
- `port` → stable internal Service port
- `targetPort` → actual app port

# Till  Day-10

[kuber-one-26.zip](Kubernetes%20notes/kuber-one-26.zip)

## Service FQDN format

```cpp
<service-name>.<namespace>.svc.cluster.local
```

# Create services for  deployments

### keyword : expose 
→ it will auto bind with deployment with labels

```bash
kubectl expose deployment deploy-ns1 \
  --name=svc-ns1 \
  --port=80 \
  --target-port=80 \
  -n ns1

```

==========================================================================

# Day-11

**Multi Container Pod Kubernetes - Sidecar vs Init Container**

[multiContainerPort.pdf](Kubernetes%20notes/multiContainerPort.pdf)

===========================================================================

# Day-12  **Daemonsets, Job and Cronjob in Kubernetes**

[Kubernetes DaemonSets Cron and job.pdf](Kubernetes%20notes/Kubernetes_DaemonSets_Cron_and_job.pdf)

===========================================================================

# #  Day-13 **Static Pods, Manual Scheduling, Labels, and Selectors in Kubernetes**

[Static-pods.pdf](Kubernetes%20notes/Static-pods.pdf)

[Manual Pod Scheduling.pdf](Kubernetes%20notes/Manual_Pod_Scheduling.pdf)

[Labels, and Selectors in Kubernetes.pdf](Kubernetes%20notes/Labels_and_Selectors_in_Kubernetes.pdf)

=======================================================================

# **Deployment Styles – Quick Summary**

### **1. Recreate**

- Stop old version → start new version
- Downtime occurs
- Simple but risky
    
    **Use:** dev, non-critical apps
    

---

### **2. Rolling Update (Default)**

- Gradual pod replacement
- No downtime
- Old & new versions run together briefly
    
    **Use:** most production services
    

---

### **3. Blue-Green**

- Two environments (Blue = live, Green = new)
- Instant switch & rollback
- Higher cost
    
    **Use:** critical systems
    

---

### **4. Canary**

- Release to small % of users first
- Monitor → increase traffic gradually
- Very safe but complex
    
    **Use:** high-risk releases
    

---

### **5. A/B Testing**

- Two versions for different user groups
- Compare behavior/metrics
    
    **Use:** feature & UX testing
    

---

### **6. Shadow (Dark Launch)**

- Traffic mirrored to new version
- Responses ignored
- No user impact
    
    **Use:** performance & integration testing
    

---

# **One-line memory hack**

> Recreate = downtime
> 
> 
> Rolling = safe default
> 
> Blue-Green = instant rollback
> 
> Canary = gradual risk
> 
> A/B = business testing
> 
> Shadow = invisible testing
> 

=========================================================================

# Day-14 **Taints and Tolerations in Kubernetes**

Taint : we add Taint  on Nodes 
Toleration  : we add Toleration on Pods

**Taints: Putting Up Fences 🚫**

Think of taints as "only you are allowed" signs on your Kubernetes nodes. A taint marks a node with a specific characteristic, such as `"gpu=true"`. By default, pods cannot be scheduled on tainted nodes unless they have a special permission called toleration. When a toleration on a pod matches with the taint on the node then only that pod will be scheduled on that node.

---

**Tolerations: Permission Slips for Pods ✅**

Toleration allows a pod to say, "Hey, I can handle that taint. Schedule me anyway!" You define tolerations in the pod specification to let them bypass the taints.

## Effect :

- NoSchedule            :  newer-pods

→ “New pods not allowed” 🚫 , BUT  OLDER PODS ARE FINE (PODS BEFORE ASSIGN TAINT)

- PreferNoSchedule  :  No-guarnatee

→ “Please don’t, but okay if needed” 😐

- NoExecute              :  existing/newer-pods

→ “Get out. Now.” 🔥 , EVEN IF YOU ARE OLDER POD , GET OUT NOW

## EFFECT IN DETAIL

### 1️⃣ NoSchedule

- Applies to **new pods only**
- Existing pods stay
- Scheduler completely blocks new pods

💡 Strict rule

---

### 2️⃣ PreferNoSchedule

- Applies to **new pods**
- Scheduler tries to avoid node
- But **no guarantee**

💡 Soft rule (Kubernetes is polite, not strict)

---

### 3️⃣ NoExecute

- Applies to **existing AND new pods**
- Existing pods get evicted
- New pods won’t be scheduled

💡 Nuclear option ☢️

### Add Taint to node

```cpp
kubectl  taint  node  <node-name>  key=value : Effect
```

### To Remove the taint added by the command above, you can run:

```cpp
kubectl taint nodes node1 key1=value1:NoSchedule-
```

### CHECK TAINT on nodes

```cpp
kubectl  describe  node  <node-name>  
//or
kubectl  describe  node  <node-name> | grep -i  taint
```

## POD-WITH-TOLERATION.yml

```cpp
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: redis
  name: redis
spec:
  containers:
  - image: redis
    name: redis
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

# NodeSelector

**NodeSelector = hard rule for scheduling**

Pod says:

> “I want nodes with this label”
> 

Node either has it… or pod stays Pending forever.

There is **no fallback**

## Hands-on Example

## Step 1: Label a node

```bash
kubectl label node worker1 env=prod
```

---

## Step 2: Pod with NodeSelector

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  nodeSelector:
    env: prod
  containers:
  - name: app
    image: nginx
```

---

## Result

- If worker1 has `env=prod` → pod runs
- If no node has it → pod stays **Pending**
- Scheduler does not argue

===========================================================================

## **Day 15/40 - Kubernetes Node Affinity Explained |**

[taintWithAffinity.pdf](Kubernetes%20notes/taintWithAffinity.pdf)

we normally use Node-affinity and Taint-TOLERATION togather 

cause if pod have tolaration with env=prod 

it goes to taint node , it will be accepted 

BUT 

IF IT GOES TO OTHER NODE 

THEY HAVE NO-PROBLEM ACCEPTING THAT 

SO 

POD CAN STILL GO TO NODE THAT ARE NOT PROD-NODE 

SO WE USE IT WITH NODE-AFFINITY 

YES 

WE CAN ALSO USE IT WITH NODE-SELECTION 

BUT AFFINITY GIVE US MORE CONTROL , SUPPORT COMPLEX-LOGIC

### HERE ARE SOME SENARIO

===========================================================================

## **Day 16/40 - Resource Requests and Limits**

[REQUEST-LIMIT.pdf](Kubernetes%20notes/REQUEST-LIMIT.pdf)

========================================================================

## **Day 17/40 - Autoscaling in Kubernetes : HPA Vs VPA**

[Autoscaling in Kubernetes -HPA Vs VPA.pdf](Kubernetes%20notes/Autoscaling_in_Kubernetes_-HPA_Vs_VPA.pdf)

HPA : HORIZONTAL POD AUTOSCALING 

VPA : VARTICAL POD AUTOSCALING 

WE MOSTLY USE HPA 
INCRESE THE NUMBER OF POD WHEN LOAD INCRESE ON POD 
→ AND DECRESE THE NUMBER OF POD WHEN LOAD DECRESE 

→ WE SET THE BAR/LIMIT LIKE IF LOAD INCRESE ON CPU AND GOSE ABOVE 65 % 

OR LOAD ON MEMORY GOES ABOVE SOME LIMIT

→ ADD ANOTHER POD 
→ AND WHEN THERE IS LESS LOAD DECRESE THE NUMBER OF POD 
→ DECRESING THE POD COUNT IS LITTLE SLOW , 
AND  WE DON’T HAVE WRITE CONDITION FOR DECRESING K8S DO IT BY IT SELF 

OR DO WE ??

ON THE OTHER HAND 

→ VPA 

WHEN LOAD INCRESE WE SHUT THE CURRENT POD 
AND CREATE A NEW POD WITH BIGGER MEMORY AND CPU 

→ BUT IT CAN TAKE TIME CLOSING ONE AND CREATING NEW ONE 

→ SO IT’D RERLY USED 

→ OR , IT HAVE SOME DIFFRENT USE CASES 

//MORE DETAILE

IF WE GO MORE DEEP 

SCALING IS NOT LIMITED TO JUST POD 

WHAT IF MEMORY OF NODE IS FINISHED AND K8S TRY TO CREATE NEW POD ON THAT 

POD WILL NOT BE CREATED CAUSE IT DON’T HAVE REQUIRED MEMORY OR CPU 

HERE COMES 

### INFRA - SCALING

→ WE INCRESE THE NUMBER OF NODES 

→ CAN’T DO THIS ON LOCAL 

→ CAUSE HERE NODE ARE DOCKER-CONTAINER 

BUT 

→ ON CLOUD EVERY NODE IS VM : VIRTUAL-MACHINE (WORKER-NODE AND CONTROL-PLANE-NODE)

SO WHEN LOAD INCRESE AND THERE IS NO SUFFICIENT MEMORY OR CPU 

→ WE SPIN-UP A NEW VM/EC2 WHICH BECOME OUR NEW-NODE 

→ NEW POD GOES INSIDE THAT WE CAN SEE THIS IN IMAGE  DETAILS

SO 

SCALING POD   ⇒ WORKLOAD-SCALING (HPA)

SCALING NODE ⇒ INFRA-SCALING (CLUSTER-AUTOSCALER)

![SCALING TYPES ](Kubernetes%20notes/image.png)

SCALING TYPES 

[hpaWithDeployment.pdf](Kubernetes%20notes/hpaWithDeployment.pdf)

### imperative command for HPA

```bash
kubectl  autoscale  deploy  <php-apache>  --cpu=50%  --min=1  --max=10
```

### declarative way

hpa.yml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

then

```bash
kubectl  apply  -f  <hpa.yml>
```

other cmd

```bash
kubectl  get hpa
kubectl  delete hpa  <hpa-name>
```

### NOTE :

here `averageUtilization: 50` 

work on RESOURCE REQUEST OF POD 

NOT ON RESOURCE LIMIT 

### NOTE :  FOR TESTING HPA , WE NEED TO INCRESE THE LOAD ON SERVER/POD

REFFER OFFICIAL DOCUMENT :

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

```bash
# Run this in a separate terminal
# so that the load generation continues and you can carry on with the rest of the steps
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

//Now run:

# type Ctrl+C to end the watch when you're ready
kubectl get hpa  <php-apache> --watch
```

Here, CPU consumption has increased to SOME % of the request. As a result, the Deployment was resized to MORE replicas:

```bash
kubectl get deployment  <php-apache> --watch
```

===========================================================================

# DAY-18 : **Kubernetes Health Probes Explained | Liveness vs Readiness Probes**

[PDF FILE](Kubernetes%20notes/Health-check-by-probe.pdf)

PDF FILE

PROBES ARE NOTHING BUT SOME SYSTEM OR SOME-PROCESS THAT CONSTANTLY LOOKING AT YOUR  SYSTEM 

OR MONITORING YOUR SYSTEM 

AND TAKING SOME NECCESSARY  ACTION  TO EITHER  RECOVER THE HEALTH 

OR TO GRAB SOME STATUS 

IN RODER TO MAKE SURE EVERYTHING WORK FINE , RUN FINE

## THEN COMES HEALTH-PROBES

TYPES OF HEALTH PROBES : 

- STARTUP
- readiness
- LIVENESS

*WE CAN USE THESE PROBES TOGATHER*

→ IF USED TOGATHER

FIRST STARTUP WILL ACTIVE 
THEN readiness + liveness (in parallel)

### LIVENESS :

→ RESTART THE APPLICATION IF FAILS 

SOMETIME YOUR APPLICATION CRASH OR SOMETHING DIFFRENT 

THAT CAN EASYLY RECOVER BY JUST RE-STARTING 

→ WE USE LIVENESS HEALTH-PROBE THERE

### readiness :

→ ENSURE THE APPLICATION IS READY

What Kubernetes does:

- If readiness = ❌
    - traffic is STOPPED
    - pod is removed from Service (IMP)
    - pod is NOT killed
- If readiness = ✅
    - traffic is allowed again

### STARTUP PROBE:

→ FOR SLOW /LAGECY APPS 

If  STARTUP  probe is **NOT passed yet**:

👉 Kubernetes does NOTHING

- It waits
- It does NOT kill the app
- It does NOT send traffic
- It just waits patiently

(no restart, no traffic, no panic)

Use   STARTUP PROBE:   ONLY when:

- you KNOW startup is slow
- you can’t fix it
- you accept the tradeoff

### THESE HEALTH PROBE CAN DO 3 TYPES OF HEALTH CHECK

- COMMAND
- HTTP REQUEST
- TCP REQUEST

===========================================================================

# DAY-19 : **kubernetes configmap and secret**

docs : [https://kubernetes.io/docs/concepts/configuration/configmap/](https://kubernetes.io/docs/concepts/configuration/configmap/)

[KubernetesConfigMap&Secret.pdf](Kubernetes%20notes/KubernetesConfigMapSecret.pdf)

===========================================================================

# DAY-21 : **Manage TLS Certificates In a Kubernetes Cluster - Create Certificate Signing Request**

[KubernetesTLSCertificates&CSR.pdf](Kubernetes%20notes/KubernetesTLSCertificatesCSR.pdf)

IMP

[Hands-on-Example.pdf.pdf](Kubernetes%20notes/Hands-on-Example.pdf.pdf)

IMP

[EXAMPLE2.pdf](Kubernetes%20notes/EXAMPLE2.pdf)

**Below document can also be referred**

[https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#create-certificatessigningreques](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#create-certificatessigningrequest)t

**TLs in Kubernetes**

![image.png](Kubernetes%20notes/image%201.png)

3 CIRTIFICATE 

- CLIENT CERTIFICATES
- SERVER CERTIFICATE
- CERTIFICATE AUTHORITY’S  CERTIFICATE  (ROOT-CERTIFICATES) (IMP)

### **Client-Server model**

![image.png](Kubernetes%20notes/image%202.png)

WHEN USER - COMMUNICATE WITH CONTROL-PLANE OR ANY COMPONENT OF CONTROL PLAN 

TEN USER  USER IS CLIENT 

AND MASTER NODE COMPONENT IS SERVER 

WE NEED ENCRYPTION THERE 

AND WHEN 

MASTER NODE COMMUNICATE WITH WORKER NODES 

MASTER NODE IS CLIENT AND WORKER NODE ARE SERVER 

WE NEED ENCRYPTION THERE 

### **How certs are loaded**

![image.png](Kubernetes%20notes/image%203.png)

### **Where we use certs in control plane components**

![image.png](Kubernetes%20notes/image%204.png)

**To generate a key file**

```
openssl genrsa -out adam.key 2048

```

**To generate a csr file FOR adam(user) | CSR = CERTIFICATE-SINGING REQUEST**

```
openssl req -new -key adam.key -out adam.csr -subj "/CN=adam"

```

**Kubernetes expects CSR in base64**

```yaml
cat adam.csr  | base64  | tr -d "\n"  # \n for removing new-line from file 
                                      # everything will be in one-line
                                      # cpoy it to next yaml file for csr object
                                      # paste it place of <BASE64_CSR_HERE>
```

 **Create Kubernetes CSR Object from csr File (private-key → csr file → csr-object)**

File: adam-csr.yaml

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: adam-csr
spec:
  request: <BASE64_CSR_HERE>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

**APPLY THAT YAML FILE TO CREATE CSR OBJECT  from Yaml (declarative)**

```yaml
kubectl  apply  -f  <adam-csr.yaml>
```

**list your CSR OBJECTS**

```yaml
kubectl  get csr  # status will be pending
kubectl describe csr  <adam-csr>
```

**To approve a csr**

```
kubectl certificate approve  <adam-csr>
```

**list your CSR OBJECTS again**

```yaml
kubectl  get csr  # status will be approve   
kubectl describe csr  <adam-csr>
```

**Convert  the approved request to Yaml**

```yaml
kubectl get csr adam-csr -o yaml > adam-issue-cert.yaml
```

**PRINT THE YAML** 

```yaml
cat adam-issue-cert.yaml
```

**copy the CERTIFICATE from  :** adam-issue-cert.yaml

```yaml
status -> certificate
```

**certificate will be in encoded formate (base64)**

**we need to decode that and put it to a file :** 

file : adam-final.cert

```yaml
echo <certificate-from-status-in-yaml> | base64 --decode > adam-final.cert
```

**now we can send this CERTIFICATE to adam (new-user) he can use this** 

**To deny a csr | CSR = CERTIFICATE-SINGING REQUEST**

```
kubectl certificate deny <adam-csr>
```

===========================================================================

# **Day 22/40 -  Authentication and Authorization**

[Authentication-vs-Authorization.pdf](Kubernetes%20notes/Authentication-vs-Authorization.pdf)

**Authentication: Verifying Who You Are**

Imagine your cluster is a fortress. Authentication is like checking IDs at the gate. Kubeconfig is your keycard containing certificates that identify you to the Kubernetes API server.

![image.png](Kubernetes%20notes/image%205.png)

**Making API Calls:**

To run a command against a cluster, you can use the below command

```
kubectl get pods --kubeconfig config
```

Normally, 

kubectl uses your local `$HOME/.kube/config` file for authentication so you dont have to pass the --kubeconfig parameter in every command. You can use below command

```
kubectl get pods
```

The below command shows an API call using raw arguments:

```
kubectl get --raw /api/v1/namespaces/default/pods \
  --server https://localhost:64418 \
  --client-key adam.key \
  --client-certificate adam.crt \
  --certificate-authority ca.crt
```

**Authorization: What You Can Do**

Authorization is like granting access levels within the fortress. Kubernetes offers different methods:

- Node Authorizer: Ensures kubelets on nodes are authorized to communicate with the API server.
- ABAC (Attribute-Based Access Control): Associates users with permissions but can be complex to manage.
- RBAC (Role-Based Access Control): The recommended approach! You create roles (like "dev") and assign users or groups to those roles.
- Webhooks (Optional): Leverage external tools like OPA for more complex authorization logic.

**Authorization Modes:**

The API server can be configured with different authorization modes (like "always allow" or "always deny"), but these are for testing only. In practice, a priority sequence is used as below:-

- Node Authorizer: Checks node communication.
- RBAC: Grants access based on assigned roles.
- Webhook (if enabled): Performs additional authorization checks.

**Remember:**

- Authentication verifies your identity.
- Authorization determines your access level.
- RBAC is a user-friendly and recommended way to manage authorization.
- Keep exploring, Kubernetes ninjas! There's more to discover in the video about configuring authentication and authorization in your cluster.

demo config-file , by default it reside in : /$HOME/.kube/config

[file : kubeConfig](file:kubeConfig) 

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: dev-cluster
    cluster:
      server: https://dev-cluster-api.example.com
      certificate-authority: /path/to/dev-ca.crt
  - name: prod-cluster
    cluster:
      server: https://prod-cluster-api.example.com
      certificate-authority: /path/to/prod-ca.crt

users:
  - name: dev-user
    user:
      client-certificate: /path/to/dev-client.crt
      client-key: /path/to/dev-client.key
  - name: prod-user
    user:
      client-certificate: /path/to/prod-client.crt
      client-key: /path/to/prod-client.key

contexts:
  - name: dev-context
    context:
      cluster: dev-cluster
      user: dev-user
      namespace: development
  - name: prod-context
    context:
      cluster: prod-cluster
      user: prod-user
      namespace: production

current-context: dev-context
preferences: {}

```

NOTE : CONTEXT = (USER + CLUSTER )

### DEFAULT AUTHORIZATION : reside in API-Server Yaml

let’s have a look

```bash
docker  ps  | grep control
#exec the control-plan 
docker exec -it  <cka-cluster2-control-plan>  bash
# visite the menifes directory (in-side control-plan)
cd  /etc/kubernetes/menifests
#list out 
ls -lrt
# print the api-server yaml file
cat  kube-apiserver.yaml
# look for the command 
# spec -> containers ->  command (array/list)
# here we can see authorization in command (array/list)
-	--autorization-mode=Node,RBAC

#Detailed
- --authorization-mode <strings>
# Ordered list of plug-ins to do authorization on secure port.
# Defaults to AlwaysAllow if --authorization-config is not used.
# Comma-delimited list of: RBAC,Node,AlwaysAllow,AlwaysDeny,ABAC,Webhook.
# MOST-USED : RBAC AND NODE (to-gather)
```

### ALL OUR CERTIFICATE  and  KEYS:

```bash
#inside control-plan container/vm
cd /etc/kubernetes/pki
ls -lrt
```

===========================================================================

# API Groups

### 1. **Core Group** (`""` or empty string)

The **core group** (sometimes referred to as the **default API group**) in Kubernetes refers to the core set of resources that are not part of any named API group. When you specify `apiGroups: [""]` in a Kubernetes RBAC rule, it means you're referring to resources in the **core group**.

The **core group** includes basic resources that are fundamental to the operation of a Kubernetes cluster, such as:

- **Pods**
- **Namespaces**
- **Nodes**
- **Services**
- **ReplicationControllers**
- **ConfigMaps**
- **Secrets**
- **Endpoints**

For example, if you want to grant permissions to interact with **pods**, you would use `apiGroups: [""]` (or simply leave it as an empty string) because `pods` belong to the core group.

**Example:**

```yaml
apiGroups: [""]
resources: ["pods"]
verbs: ["get", "list", "watch"]

```

Here, `""` indicates that you're referring to the **core API group**, and `pods` are one of the core resources.

### 2. **Named Group**

A **named group** refers to any API group other than the core group. These groups are generally used for more specialized or extended functionality, such as resources for extensions, batch jobs, or networking.

A **named API group** is identified by a string, like `"apps"`, `"batch"`, `"networking.k8s.io"`, `"rbac.authorization.k8s.io"`, etc. These groups allow Kubernetes to support different types of resources, such as:

- **"apps"**: Includes resources like Deployments, StatefulSets, and ReplicaSets.
- **"batch"**: Includes resources like Jobs and CronJobs.
- **"networking.k8s.io"**: Includes resources like NetworkPolicies and Ingresses.
- **"rbac.authorization.k8s.io"**: Includes resources related to RBAC (Roles, RoleBindings, ClusterRoles, etc.).

When you specify an **apiGroup** with a name other than `""`, you're referring to resources in one of these named API groups.

**Example:**

```yaml
apiGroups: ["apps"]  # Refers to the 'apps' API group
resources: ["deployments"]
verbs: ["get", "list"]

```

In this case, you’re referring to resources in the **"apps"** group, and specifically the **deployments** resource.

### Summary:

- **Core Group (`""`)**: Refers to resources that are built into the default Kubernetes API. These resources don't have a specific group name, like **pods**, **services**, **namespaces**, and so on.
- **Named Group**: Refers to any API group that is named and includes extended resources. Examples are the **"apps"** group (with resources like **Deployments**), **"batch"** (with **Jobs** and **CronJobs**), and **"rbac.authorization.k8s.io"** (for **Roles**, **RoleBindings**, and related resources).

The `apiGroups` field in Kubernetes RBAC allows you to define the scope of your rule, specifying whether you're granting access to core resources or extending it to a specific named API group.

===========================================================================

# **Day 23/40 -**

## **Kubernetes RBAC Explained - Role Based Access Control Kubernetes**

[RBAC.pdf](Kubernetes%20notes/RBAC.pdf)

```bash
kubectl auth can-i get pod
#result : yes | no

kubectl auth can-i get pod --as <adam-new-user>
#result : yes | no

kubectl auth whoami
# result : 
# ATTRIBUTE                       VALUE
# Username                        kubernetes-admin
# Groups                          [kubeadm:cluster-admins system:authenticated]
```

### CREATING ROLE OBJECT

File : Role.yaml

```yaml
# Define the API version and kind of the resource (Role)
apiVersion: rbac.authorization.k8s.io/v1  # API version for RBAC roles
kind: Role  # The kind of Kubernetes resource, in this case, a Role

# Metadata to identify and describe the Role
metadata:
  namespace: default  # The namespace where this role applies (default in this case)
  name: pod-reader    # The name of the Role

# Define the rules that specify what this Role allows
rules:
  # A list of rules for different resources
  - apiGroups: [""]  # "" refers to the core API group (for basic resources like pods)
    
    # Specify which resources this role grants access to
    resources: ["pods"]  # This Role grants access to "pods" resources
    
    # Specify what verbs (permissions) are allowed on the resource
    verbs: 
      - "get"    # Allows reading the details of pods
      - "list"   # Allows listing all pods in the namespace
      - "watch"  # Allows watching for changes to pods in real-time

```

THEN 

```bash
kubectl apply -f  Role.yaml
```

OTHER RELATED COMMAND

```bash
kubectl get roles

kubectl get roles -A  # from all namespaces

kubectl  describe role  <pod-reader>
```

Q. Command for count the number of roles (all name-space)

```bash
kubectl get roles -A --no-headers | wc -l
```

HERE WE JUST CREATED ROLE 

WE HAVE TO ASSIGN THIS ROLE TO USER 

SO 

IT’S CALLED ROLE-BINDING 

→ ROLE-BINDING  ALSO DONE USING YAML FILE ( DECLARATIVE-WAY )

### ROLE-BINDING

**Sample YAML for role binding**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: krishna # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
  
```

OTHER RELATED COMMAND 

```bash
kubectl get rolebinding
kubectl describe rolebinding  <read-pods>
```

NOW TRY THESE

```bash
kubectl auth can-i get pod --as adam
# result : yes

kubectl auth can-i delete pod --as adam
# result : no
```

CURRENTLY WE ARE JUST CHECKING CAN WE RUN THIS COMMAND AS  <ADAM-NEW-USER>

→ BUT WE HAVEN’T LOGGED IN AS  <ADAM-NEW-USER>

SO

HOW WE CAN LOGIN AS ADAM

HOW WE CAN RUN THE COMMAND AS ADAM

→ FOR THAT WE NEED TO ADD ENTRY IN KUBE-CONFIG

```bash
kubectl config set-credentials adam \
--client-key=../day-21/adam.key \
--client-certificate=../day-21/adam-final-certificate \
--embed-certs=true
```

NOW SET THE CONTEXT

context = (user + cluster)

```bash
kubectl config set-context <adam>  --cluster=kind-cka-cluster1 \
--user=adam 

#Result : Context "adam" created.
# new Context will be created 

kubectl config use-context  <adam>
# result : Switched to context "adam".

kubectl config  get-context

kubectl auth whoami
#result
# ATTRIBUTE                       VALUE
# Username                        adam
```

BUT THIS CAN SHOW ERROR (SOMETIME)

WHY ??

→ CERTIFICATE  EXPIRED (in our example expire=1-day)

try this :

```bash
openssl x509 -noout -dates -in ../day-21/adam-final-certificate

#result
notBefore=Feb  2 08:46:24 2026 GMT
notAfter=Feb  3 08:46:24 2026 GMT
```

NOW ADAM-USER HAVE ACCES TO LIST PODS

BUT TRY FOR DEPLOY : 

```bash
kubectl get deploy

#Result : 
# Error from server (Forbidden): deployments.apps is forbidden:
# User "adam" cannot list resource "deployments" in API group "apps" in the namespace "default"
```

OTHER IMPORATNT COMMAND 

```bash
kubectl config view
#FOR CURRENT-CONTEXT
# CURRENT USER 
# CURRENT CLUSTER
```

===========================================================================

# DAY 24/40 : **RBAC Continued - Clusterrole and Clusterrole Binding**

[CR&CRB.pdf](Kubernetes%20notes/CRCRB.pdf)

PREVIUSLY WE SEE ROLE AND ROLE BINDING 

THEY ARE NAMESPACE SCOPE LEVEL 

MEANS ALL THEIR CAPABILITY LIMITED TO NAMESPACE PROVIDED

Q. WHAT RESOURCE ARE APPLICABLE TO NAMESPACE 

- PODS
- DEPLOYMENT , RS
- SERVICES

### NOW WE GOING FOR CLUSTER-ROLE AND CLUSTER-ROLE-BINDING

Q. WHAT RESOURCE ARE APPLICABLE TO  CLUSTER ??

- NAMESPACE (YES , IT IS)
- NODES
- ETC..

FOR CLUSTER ROLE-BINDING

WE CAN ATTACH IT TI TO USER OR GROUP 

Q. GET  ALL THE RESOURCE  AT NAMESPACE LEVEL

```bash
kubectl api-resources --namespaced=true
```

Q. GET ALL THE RESOURCE AT CLUSTER LEVEL (--namespaced=FALSE)

```bash
kubectl api-resources --namespaced=false
```

TRY THIS

```bash
kubectl auth can-i get nodes --as adam
# RESULT :

# Warning: resource 'nodes' is not namespace scoped
# no
```

SO WE NEED TO CREATE THE PERMISSION

### IMPERATIVE WAY

```bash
kubectl create clusterrole --help
```

CHECK : USAGE 

IT HAVE A SAMPLE COMMAND 

COMMAND - STRUCTURE

```bash
kubectl create clusterrole NAME --verb=verb --resource=resource.group \
   [--resource-name=resourcename] \
   [--dry-run=server|client|none] [options]
```

EXAMPLE : 

```bash
kubectl create clusterrole <read-node-cr> \
--verb=list,get,watch \
--resource=node

//or

kubectl create clusterrole read-node-cr  \
--verb=list,get,watch \
--resource=node --dry-run=client -o yaml > cr2.yaml
```

OTHER-COMMAND

```bash
kubectl get clusterrole

kubectl describe clusterrole <read-node-cr>
```

### DECLARETIVE WAY

FILE : cr.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-node-cr
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - get
  - watch
```

ROLE HAS BEEN CREATED 

NOW WE NEED TO ATTACH THIS CLUSTER ROLE TO USER OR GROUP

WE DO THIS BY  CLUSTER-ROLE-BINDING

 

### imperative way

```bash
kubectl create clusterrole --help

kubectl create clusterrolebinding NAME --clusterrole=NAME \
[--user=username] [--group=groupname] \
[--serviceaccount=namespace:serviceaccountname] \
[--dry-run=server|client|none] [options]

kubectl create clusterrolebinding  <read-node-crb> \
--clusterrole=<read-node-cr> \
--user=<adam>

//or
kubectl create clusterrolebinding  read-node-crb \
--clusterrole=read-node-cr \
--user=adam --dry-run=client -o yaml > crb.yaml
```

### declarative way

FILE: crb.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-node-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: read-node-cr
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: adam
```

```bash
kubectl get clusterrolebinding 
kubectl describe clusterrolebinding <read-node-crb>
```

NOW CHECK 

```yaml
kubectl auth can-i get nodes --as=adam

# RESULT :
# yes
```

CURRNETLY WE HAVE READ ACCES 

TRY DELETE NODES 

```yaml
kubectl delete nodes cka-cluster1-worker
# Error from server (Forbidden):
# nodes "cka-cluster1-worker" is forbidden: User "adam" cannot delete resource "nodes" in API group "" at the cluster scope
```

===========================================================================

# **DAY-25/40 RBAC Continued - Service Account**

SERVICE ACCOUNT 

IT IS NOT USUALLY USED BY HUMAN USERS 

IT IS MOSTLY USED BY POD 

SERIOSLY POD ??

YES , IT’S MOSTLY USED BY PODS 

[SERVICE-ACCOUNT.pdf](Kubernetes%20notes/SERVICE-ACCOUNT.pdf)

- Create a service account
- Create a role
- Bind the role to the service account (role binding)
- Finally, attach the service account to the  Pods

NOW EXEC POD 

AND WE CAN RUN `kubectl get pods`

BUT IT WILL NOT WORK DIRECTLY WE NEED TO INSTALL KUBECTL INSIDE POD 

OR USE KUBECTL IMAGE FROM BITNAMI (ADD THE SERVICE ACCOUNT)

EXAMPLE : 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubectl-pod
spec:
  serviceAccountName: my-sa
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "infinity"]
```

Q. WHY PODS NEEDS ACCOUNT 

OR EVEN WHY POD NEED ACCES OVER ANY OTHER RESOURCE ??

→ POD’S SOMETIME NEED TO LIST ALL PODS AND OTHER RESOURCE

EXAMPLE : JENKINS SERVER , AGRO-CD , prometheus (MONITORING-TOOL) , DATA-DOG , ,ETC…

IF NOT SERVICE ACCOUNT , THEY DEFAULT USE ADMIN ACCOUNT 

THAT CAN BE RISKY 

FOR DEEP-DETAILS READ BELOW PDF : 

[WHY-SA.pdf](Kubernetes%20notes/WHY-SA.pdf)

https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-long-lived-api-token-for-a-serviceaccount

```yaml
kubectl get sa

kubectl create sa build-sa

kubectl describe sa build-sa
```

FILE : secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-sa
type: kubernetes.io/service-account-token
```

```yaml
kubectl apply -f secret.yaml
kubectl describe secret build-robot-secret
```

```yaml
kubectl auth can-i get pods --as build-sa

kubectl create role build-role \
--verb=list,get,watch \
--resource=pod

kubectl create rolebinding rb \
--role=build-role \
--user=build-sa

```

===========================================================================

# DAY 26 : NETWORK POLICY (CNI)

[NetworkPolicy.pdf](Kubernetes%20notes/NetworkPolicy.pdf)

calico CNi : kubectl apply -f [https://docs.projectcalico.org/manifests/calico.yaml](https://docs.projectcalico.org/manifests/calico.yaml)

check : k get pod -A

==========================================================================

# DAY-28 DOCKER VOLUME

IMP  : BIND MOUNT  VS VOLUME BIND

VOLUME-DRIVER VS STORAGE DRIVER 

PERSISTANT STORAGE VS NON-PERSISTENT STORAGE 

===========================================================================

# DAY-29 K8S STORAGE

[k8s-Storage.pdf](Kubernetes%20notes/k8s-Storage.pdf)

===========================================================================

# DAY 30 - DNS

[dns.pdf](Kubernetes%20notes/dns.pdf)

==========================================================================

# DAY-31 **CoreDNS In Kubernetes**

[CORE-DNS.pdf](Kubernetes%20notes/CORE-DNS.pdf)

→ it provide the FUNCTIONALITY OF DOMAIN NAME RESOLUTION WITH THE SERVICE TO IP NAME 

[https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
 

===========================================================================

# Day 32/40 - Kubernetes Networking  | Container Network Interface (CNI)

OCI : OPEN CONTAINER INITIATIVE

https://github.com/saiyam1814/kube-networking?tab=readme-ov-file

==========================================================================

# **Day 33/40 - Kubernetes Ingress Tutorial**

Option 1: NodePort

Problems:

- Ugly URLs
- Random ports
- Hard to remember
- Not production-friendly

Option 2: LoadBalancer

Problems:

- One LoadBalancer per service
- Expensive
- Not scalable

## 💣 Problem

We need:

- One entry point
- Clean URLs
- Domain-based routing
- Path-based routing
- TLS support

Example desired behavior:

```
app.com/api      → backend service
app.com/frontend → frontend service

```

NodePort  and  LoadBalancer  alone **can’t do this cleanly**.

![image.png](Kubernetes%20notes/image%206.png)

But important truth:

> **Ingress is only a rule.
Ingress Controller is the actual router.**
> 

# Step 1 – Core Components

Ingress setup has **3 parts**:

| Component | Purpose |
| --- | --- |
| Service | Exposes pods internally |
| Ingress | Routing rules |
| Ingress Controller | Actual traffic handler |

Common controllers:

- NGINX Ingress (most popular)
- Traefik
- HAProxy
- Istio Gateway
- AWS ALB

# Step 2 – How traffic flows

```
User → Ingress Controller → Ingress Rules → Service → Pod
```

Example:

```
app.com/api → backend service
```

[custom-Ingress.pdf](Kubernetes%20notes/custom-Ingress.pdf)

FILE : ingress.yaml  , imp-part : Rule(list) and ingressClassName

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: "example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
  - host: "xyz.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-pod1
            port:
              number: 80
```

===========================================================================

# DAY-35 **ETCD Backup And Restore**

===========================================================================

# DAY-36 **Logging and Monitoring**

===========================================================================

# DAY-43 HELM AND HELM CHART

https://github.com/iam-veeramalla/helm-zero-to-hero/

helm is package manager for KUBERNETES 

LIKE APT FOR UBUNTU,DEBIAN

NPM FOR NODE , PIP FOR PYTHON , ETC…

→ IT LET US INSTALL (SPECIFIC-VERSION)

→ UPDATE (TO SPECIFIC VERSION)

→ REMOVE (UNINSTALL)  PACKAGES 

WE NEDD TO INSTALL CONTROLLER IN K8S 

LIKE PROMETHIUS ,GRAFANA , ARGOCD , NGINX-INGRESS, ETC..

### IMP

→ IT ALLOW TO BUNDLE APPLICATION OF OUR ORGANIZATION 

AS HELM CHART 

### INSTALLING HELM

→ WE INSTALL IT ON MACHINE WITH APT 

→ WE DO NOT NEED IT TO INSTALL ON CLUSTER 

install-command 

```yaml
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

add repo 

→ we connect a jounral repo with help , all wellknown package available there

→ if we bundle(chart) our own organization application 

→ and we need to install it somewhere else

→ we first need to connect helm with our repo 

command :

```yaml
helm repo add bitnami https://charts.bitnami.com/bitnami
```

→ BITNAMI IS ONE-OF THE MOST FAMUS REPO FOR CHART 

### Imp : BUNDAL = CHART

Q. WE HAVE MULTIPAL CLUSTER , DOES IT INSTALL PACKAGE(CHART) TO ALL MY CLUSTER  , OR IT ASK FOR NAME OF SPECIFIC CLUSTER ??

→ LIKE KUBECTL , HELM ALSO INSTALL/APPLY/CREATE IN CURRENT-CONTEXT 

RELESE : CUSTUM NAME OF THE CHART (pakage-bundle)

```yaml
helm install custom-nginx bitnami/nginx

kubectl get deploy //deploy-name= custom-nginx
kubectl get pods   //pod-name = custom-nginx-xysjhj

helm  list

helm  uninstall custom-nginx
```

### create chart

```yaml
helm create mysql-chart
```

→ it will create a Folder with values and templates 

→ templete have all the required yaml 

→ eg. deployment , service , serviceaccount , hpa , ingress

→ these template have variable , which take values from values.yaml

IMP : NO MATTER WHAT NAME YOU USE , HELM CREATE DEFAULT CHART FOR NGINX

→ WE CAN CHACK OR CHANGE FROM VALUE.YAML
