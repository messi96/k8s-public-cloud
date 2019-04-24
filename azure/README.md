AKS (Azure Kubernetes Services)

	Basic Ingress

		https://docs.microsoft.com/en-us/azure/aks/ingress-basic

	Ingress TLS 

		https://docs.microsoft.com/en-us/azure/aks/ingress-tls

1. Install flexvolume driver as daemonset on your cluster.

	Add the add-on "keyvault-flexvolume" to automatically enable key vault flexvolume in your k8s cluster.

		https://github.com/Azure/aks-engine/blob/master/examples/addons/keyvault-flexvolume/README.md

		kubectl create -f kv-flexvol-installer.yaml
	flexvol driver can be installed on any existing namespace.
	The driver itself is implemented as daemonsets, so you'll end up having one keyvault pod per physical node in the cluster.

2. Create a namespace for your application

		kubectl create -f namespace.yaml

3. Cluster should have helm/tiller installed

4. Install Ingress controller with static Public IP address. (or random allocation)				(needed for https configuring)
	
	get the resource-group name of your cluster
		
		az aks show --reousrce-group rg-name --name cluster_name --query nodeResourceGroup -o tsv			#var

	create a public IP address with static allocation method

		az network public-ip create --resource-group var --name myakspublicip --allocation-method-static
	
	This will give you the static public IP address say : 11.11.11.11

	Use this address as the ip for your ingress controller

	install nginx-ingress controller

		helm install . --namespace app --name nginx-ingress --set controller.replicaCount=2 --set controller.service.loadBalancer=11.11.11.11 --tiller-namespace=kube-system -f values.yaml

		sh$ kubectl get service -l app=nginx-ingress
		NAME				TYPE			CLUSTER-IP			EXTERNAL-IP			PORT(s)				AGE
		nginx-ingress-controller	LoadBalancer		10.0.0.122			11.11.11.11			80:32323/TCP,443:32333/TCP	1d		
		nginx-ingress-default-backend	ClusterIP		10.0.0.129			<none>				80/TCP				1d


5. Configure a DNS name												(needed for https configuring)

		./dns-map.sh

		sh$ az network public-ip update --ids /subscriptions/xxxxxxxxxx/resourceGroups/var/providers/Microsoft.Network/publicIPAddresses/myakspublicip --dns-name demo-app
		{
		  "dnsSettings" : {
		     "domainNameLabel" : "demo-app"
		     "fqdn" : "demo-app.westus.cloudapp.azure.com"
		    .... ..
		    ....
	
	DNS name "demo-app" is configured
	with FQDN :  demo-app.westus.cloudapp.azure.com
	The ingress controller is now accessible through the FQDN.

6. Install cert-manager to provision & manage TLS certificates

		# Install the CustomResourceDefinition resources separately
		kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml

		# Create the namespace for cert-manager
		kubectl create namespace cert-manager

		# Label the cert-manager namespace to disable resource validation
		kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

		# Add the Jetstack Helm repository
		helm repo add jetstack https://charts.jetstack.io

		# Update your local Helm chart repository cache
		helm repo update

		# Install the cert-manager Helm chart
		helm install \
		  --name cert-manager \
		  --namespace cert-manager \
		  --version v0.7.0 \
		  jetstack/cert-manager

		#helm install through local chart
`		helm install . --namespace add --name cert-manager --tiller-namespace kube-system -f values.yaml

7. Create a cluster Issuer

	Before certificates can be issued, cert-manager requires an issuer or ClusterIssuer resource.
	https://cert-manager.readthedocs.io/en/latest/reference/issuers.html

		kubectl create -f cluster-issuer.yaml
	Issuers (and ClusterIssuers) represent a certificate authority from which signed x509 certificates can be obtained, such as let's encrypt.
	You will need at least one Issuer or ClusterIssuer in order to begin issuing certificates within your cluster.
	Currently using staging.

8. Setup config files for the application

	Configure a azure disk file storage from the available options under yor rg and subscription.

		sh$ az disk list --resource-group $var --subscription=xxxxxxxx

		......

		...
		"id" : "/subscriptions/xxxxx/resourceGroups/rg-name/providers/Microsoft.compute/disks/application_logs"
		....

	application_logs is the azure disk managed service created.

	we have the resource id which should be used in PV & PVC 
	mount the azure disk & flexvol driver to retrieve keyvault secrets, certs during the runtime.
	volume after mounting by default have only root owning permissions, so we have two options to get around that :

		- securitycontext 	fsgroup
		- initcontianers 	initcontainers to change the owner permissions for the mounted volume.

	
		kubectl create -f pv.yaml    pvc.yaml    

		kubectl create secret docker-registry reg-cred --docker-registry="myhub.io" --docker-username="" --docker-password="" --docker-email=""
		kubectl create secret generic kvcreds --from-literal clientid="xxx" --from-literal clientsecret="xxxxx" --type="azure/kv"

	
	regcred : to reach the private registry
	kvcreds : to retrieve secrets from keyvault

	vault creation :
		
		az keyvault create --location=westus --name myvault --resource-group rg_name
	
	secrets to vault :
	
		az keyvault secret set --vault-name myvault --name keystore --value @keystore.jks

	Service can be deployed as LB/ClusterIP, our ingress-controller will take care of it.

9. Create an Ingress route

	To access our application from public network, create an ingress resource.
	The ingress resource configures the rules that route traffic to one of the two applications.
	Make sure of the annotations used correctly  as per the ingress-controller active.

		kubectl create -f ingress.yaml

10. Verify Certificate installation

		kubectl create -f certificates.yaml


Done!!!
