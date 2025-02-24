---
layout: post
title:  "1st Project with my Homelab: Strava Data in Grafana Deployment "
date:   2025-02-24 17:44:00 +0800
categories: k8s, helm, prometheus, grafana, strava, ingress, TLS, DNS
---

Step 1: Install Docker
	Ensure Docker is installed on your machine. You can download Docker Desktop from the official Docker website.
	https://docs.docker.com/desktop/setup/install/linux/ubuntu/

Step 2: Enable Kubernetes in Docker Desktop
	Open Docker Desktop.
	Navigate to Settings.
	Select the Kubernetes tab.
	Toggle Enable Kubernetes and click Apply & Restart1.
  
Step 3: Install kind & kubectl
	kind (Kubernetes IN Docker) is a tool for running local Kubernetes clusters using Docker container nodes. Install kind by following these steps:

Install kind using Go:
 
		$ GO111MODULE="on" go get sigs.k8s.io/kind@v0.11.1
  
Verify the installation:
 
		$ kind --version

	Install kubectl on local machine
  refer: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

	Set alias & autocomplete (put inside ~/.bashrc)
 
		alias k=kubectl
		source <(kubectl completion bash)
		source /usr/share/bash-completion/bash_completion
		complete -o default -F __start_kubectl k


Step 4: Create a Kubernetes Cluster with kind
	Create a configuration file for your cluster, e.g., kind-config.yaml:
 
		kind: Cluster
		apiVersion: kind.x-k8s.io/v1alpha4
		nodes:
  		- role: control-plane
  		- role: worker
  		- role: worker
    
Create the cluster:
	kind create cluster --config kind-config.yaml
  
Step 5: Verify the Cluster
	Check the status of your cluster:
 
		kubectl cluster-info
    
Step 6: Deploy Applications
	You can now deploy your applications to the Kubernetes cluster using kubectl commands. For example, to deploy a simple Nginx application:

		kubectl create deployment nginx --image=nginx
		kubectl expose deployment nginx --port=80 --type=NodePort
    
Step 7: Access Your Application
	Find the NodePort assigned to your service:

		kubectl get services
  
Access your application using http://localhost:<NodePort>.


Step 8: Setup Helm for prometheus and grafana

1: Install Helm
Helm is a package manager for Kubernetes that simplifies the deployment process. If you don't have Helm installed, you can install it by following the official Helm installation guide.

2: Add Helm Repositories
Add the Prometheus and Grafana Helm repositories:

	helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
	helm repo add grafana https://grafana.github.io/helm-charts
	helm repo update
  
3: Deploy Prometheus
Create a namespace for monitoring:

	kubectl create namespace monitoring
   
Install Prometheus using Helm:
 
	helm install prometheus prometheus-community/prometheus --namespace monitoring
      
4: Deploy Grafana
Install Grafana using Helm:

	helm install grafana grafana/grafana --namespace monitoring
   
Retrieve the Grafana admin password:
 
	kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   
Forward the Grafana port to access the dashboard:

	kubectl port-forward --namespace monitoring svc/grafana 3000:80
   
You can now access Grafana at http://localhost:3000 and log in with the username admin and the password you retrieved.

5: Configure Grafana to Use Prometheus

* Log in to Grafana.
* Add Prometheus as a data source:
* Go to Configuration > Data Sources.
* Click Add data source.
* Select Prometheus.
* Set the URL to http://prometheus-server.monitoring.svc.cluster.local:80.
* Click Save & Test.

6: Create Dashboards
You can now create dashboards in Grafana to visualize the metrics collected by Prometheus. Grafana provides many pre-built dashboards that you can import and customize.

Step 9: Connect with Strava
  Step 1: login to your strava on browser
    https://www.strava.com/settings/api
      
    create apps
      
	Step 2: install strava plugin at your grafana
    grafana>connections>add new connection>strava >install > add new data
  	fill in the credential based on the strava api
    
    Client ID: <Client_ID>
    Client Sec: <Client_Sec>

Step 10: To make your Grafana dashboard accessible via a URL with Ingress, TLS, and DNS, follow these steps:

1: Install and Configure Ingress Controller

Install NGINX Ingress Controller:

	kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

Verify the Ingress Controller:

	kubectl get pods -n ingress-nginx

2: Create Ingress Resource for Grafana

Create an Ingress resource:

	apiVersion: networking.k8s.io/v1
	kind: Ingress
	metadata:
	  name: grafana-ingress
	  namespace: monitoring
	  annotations:
	    nginx.ingress.kubernetes.io/rewrite-target: /
	    cert-manager.io/cluster-issuer: "letsencrypt-prod"
	spec:
	  tls:
	  - hosts:
	    - grafana.example.com
	    secretName: grafana-tls
	  rules:
	  - host: grafana.example.com
	    http:
	      paths:
	      - path: /
	        pathType: Prefix
	        backend:
	          service:
	            name: grafana
	            port:
	              number: 80
              
Apply the Ingress resource:

	kubectl apply -f grafana-ingress.yaml

3: Set Up TLS with Cert-Manager

Install Cert-Manager:

	kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.yaml

Create a ClusterIssuer for Let's Encrypt:

	apiVersion: cert-manager.io/v1
	kind: ClusterIssuer
	metadata:
	  name: letsencrypt-prod
	spec:
	  acme:
	    server: https://acme-v02.api.letsencrypt.org/directory
	    email: your-email@example.com
	    privateKeySecretRef:
	      name: letsencrypt-prod
	    solvers:
	    - http01:
	        ingress:
	          class: nginx
          
Apply the ClusterIssuer:

	kubectl apply -f cluster-issuer.yaml

4: Configure DNS

Create a DNS record for grafana.example.com pointing to the external IP of your Ingress controller. This can be done through your DNS provider's management console.

5: Access Grafana Dashboard

Once everything is set up, you can access your Grafana dashboard securely at https://grafana.example.com.

Additional Notes

TLS Configuration: Ensure that the grafana-tls secret is created by Cert-Manager and contains the necessary certificates.
DNS Propagation: It may take some time for DNS changes to propagate.
    
