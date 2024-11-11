# Kubernetes Security Implementation
The goal of the kubernetes Security Implementation project is to establish a secure, compliant, and resilient kubernetes environment by proactively addressing potential security vulnerabilities and acces control risks.


Project 3: KUBERNETES SECURITY IMPLEMENTATION

To implement a robust security approach in a kubernetes environment, here's a comprehensive, real-world step-by-step guide to help me set up, secure, and monitor my kubernetes cluster. These instructions will prepare me not only to secure Kubernetes but also to teach others with confidence.


   1- I set up a kubernetes cluster with Minikube or AWS EKS

I start by setting up a kubernetes environment. For production-like environements, AWS EKS is recommended, but for learning and local development, Minikube will work well.

   1.1 I set up Minikube (for local testing)
    
   Step 1: I install Minikube and kubectl.
   -> curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

    On macOs:
    -> brew install minikube
    
    Install Kubectl:
    -> curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/$(uname | tr '[:upper:]' '[:lower:]')/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
     
     Step 2: I start Minikube with a specific driver.
     
     -> minikube start --driver=docker
     
     1.2- I set up AWS EKS (production environment)
 To set up eks, i will need an AWS account. AWS provides an eksctl tool for simplifying this setup.
 
     1. I install eksctl:
     -> curl --location "https://github.com/weaveworks/eksctl/releases/download/$(curl --silent "https://api.github.com/repos/weaveworks/eksctl/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

     
     2- I create the EKS cluster:
    -> eksctl create cluster --name my-secure-cluster --region us-west-2 --nodegroup-name standard-workers --node-type t3.medium --nodes 3
    
    
      2- I configure Role-Base Acess control (RBAC)
 RBAC controls who can access what in my cluster. Proper RBAC is essential for limiting access and following the principle of least privilege.
 
      2.1 - I understand Kubernetes RBAC Components
      
      * Roles: Define a set of permissions (e.g.,view,edit) within a namespace.
      * ClusterRoles: Define permissions cluster-wide.
      * RoleBinding and ClusterRoleBindings: Associate Roles with users, groups or service accounts.
      
      
      2.2 - I create an example Role and RoleBanding.
 Let's create a role for a "Developer" to view resources within the "dev" namespace.
 
      
       Step 1: I create a new namespace.
       -> kubectl create namespace dev
       
       step 2: I define a role in YAML (developer-role.yaml):
       
 apiVersion: rbac.authorization.k8s.io/v1
 kind: Role
 metadata:
     namespace: dev
     name: developer
 rules:
 -  apiGroups: [""]
    resources: ["pods", "services", "deployments"]
    verbs: ["get", "list", "watch"]
    
    
    Apply the role:
    
    -> kubectl apply -f developer-role.yaml
    
    
    step 3: I create a RoleBanding to associate this role with a specific user or service account (developer-rolebanding.yaml):
    
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Rolebanding
    metadata:
      name: developer-banding
      namespace: dev
    subjects:
    -   kind: User
        name: "developer-user" # replace with user
        apiGroup: rbac.authorization.k8s.io
    roleRef:
       kind: Role
       name: developer
       apiGroup: rbac.authorization.k8s.io
       
     
     Apply the Rolebinding:
     -> kubectl apply -f developer-rolebinding.yaml
     
     
     3 - I install a security scanner for runtime Monitoring
 For real-time security, tools like Falco or Aqua Security are valuable
 
     3.1 I set up Falco for runtime Monitoring
  Falco is an open-source kubernetes runtime security tool that detects suspicious activity in real time.
  
    step 1: I install Falco via Helm
    
  -> curl --location "https://github.com/weaveworks/eksctl/releases/download/$(curl --silent "https://api.github.com/repos/weaveworks/eksctl/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
  
  Note: Falco agent are spinning up on each node in my cluster. After a few seconds, they are going to start monitoring my container looking for issues.
  
  
  Step 2: I configure Falco to monitor key activies. I can define custome rules, but Falco includes useful defaults like detecting abnormal process containers.
  
  Step 3: I verify Falco is running.
  -> kubectl get pods -n falco
  -> kubectl logs <falco-pod-name> -n falco
  
  
  4 - I implement Pod Security Policies (PSP) and Network poliicies
  
  4.1 - POd Security Policies
Note: PSPs are deprecated as of kubernetes 1.21. instead, use Open Policy Agent (OPA) or Kyverno.

   step 1: I define a PSP that restricts certain actions (restricted-psp.yaml):
   
   apiVersion: policy/v1beta1
   kind: PodSecurityPolicy
   metadata:
      name: restricted
   spec:
     privileged: false
     allowPrivilegeEscalation: false
     runAsUser:
         rule: MustRunAsNonRooT
     selinux:
         rule: RunAsAny
     supplementalGroups:
         rule: MustRunAs
         ranges:
         -  min: 1
            max: 655355
     fsGroup:
        rule: MustRunAs
        ranges:
        -   min: 1
            max: 65535
     volumes:
     -  'configMap'
     -  'secret'
     -  'emptyDir'
     
  Apply the policy
  -> kubectl apply -f restricted-psp.yaml
  
  note: i have to troubleshoot this part cause there is no match.ubectl apply -f restricted-psp.yaml 
error: unable to recognize "restricted-psp.yaml": no matches for kind "PodSecurityPolicy" in version "policy/v1beta1"

   
    4.2 - Network Policies
 Network policies restrict traffic between pods, enforcing security at the network level.
 
    
    step 1: I define a network policy (allow-fontend.yaml) to allow frontend pods to talks to backend pods in the same namespace.
    
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
       name: allow-frontend
       namespace: dev
    spec:
       podSelector:
          matchLabels:
             role: Backend
       ingress:
       -  from:
          -  podSelector:
                 matchLabels:
                    role: frontend
             policyTypes:
             - Ingress
             
      Apply:
      -> kubectl apply -f allow-frontend.yaml
      
     
     
     5 - I set up Audit Logs and Alerts
     5.1 I enable Kubernetes Audit logs
     
Audit logging provides insights into actions within the cluster, capturing events for security and troubleshooting.

     step 1: I Create an audit-policy.yaml file to define what events to log.
     
     apiVersion: audit.k8s.io/v1
     kind: Policy
     rules:
       - level: Metada
         verbs: ["create", "update", "delete"]
         resources:
         - group: ""
           resources: ["pods", "services"]
          namespace: ["default"]
          
       
      step 2: I enable auditing by updating the API server configuration (add --audit-policy-file=/path/to/audit-policy.yaml).
      
      
      5.2- I set Up Alert with Cloudwatch (if using EKS)
 For aws eks:
 
      1- I use Cloudwatch container insights to collect and monitor logs.
      2- I set Up cloudwatch alarms based on log patterns (e.g., failed access attempts or suspicious activity).
      
      
     For Minikube or ON-Prem:
    1- I use Prometheus Alertmanager to trigger alerts on custom log queries or integrate with a SIEM
    
    
    Final STEPS AND BEST PRACTICES:
 * Regular review Policies: RBAC and PSP configurations should be reviewd periodically.
 
 * Automate Security Scanning: Integrate tools like Falcoo and Aqua Security in CI/CD pipelines.
 
 * Document procedures: keep documentation for my team on how to respond to incidents and monitor security.
 
 * Monitor for Updates: Security tools often update rules and policies to handle new vulneabilities.
     
