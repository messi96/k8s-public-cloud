Wordpress	

kubectl create -f namespace.yaml  secrets.yaml  deploy.yaml  service.yaml

	root@pwned:~/gcp/k8s-public-cloud/gcp/wordpress# kubectl get pods,svc
	NAME                                        READY   STATUS    RESTARTS   AGE
	pod/wordpress-deployment-7b954b495d-rsc42   2/2     Running   0          5m52s

	NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)           AGE
	service/wordpress-service   LoadBalancer   10.31.245.183   35.200.228.95   31001:31001/TCP   111s

Our service is now exposed outside the cluster @ http://35.200.228.95:31001 


Data is not persisted, so if our pod restarts everything is lost!

type:nodeport or loadbalancer
nodeport will open the port on every node of the cluster, but since in gcp nodes are not reachable directly from internet we have to deploy a basic ingress controller that will route the service to the external public facing domain name.

kubectl create -f ingress.yaml

	root@pwned:~/gcp/k8s-public-cloud/gcp/wordpress# kubectl get pods,svc,ing
	NAME                                        READY   STATUS    RESTARTS   AGE
	pod/wordpress-deployment-7b954b495d-rsc42   2/2     Running   0          25m

	NAME                        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
	service/wordpress-service   NodePort   10.31.245.183   <none>        31001:31001/TCP   21m

	NAME                               HOSTS   ADDRESS        PORTS   AGE
	ingress.extensions/basic-ingress   *       34.96.94.181   80      36s


Now our application should be accessible from 34.96.94.181 directly, which is a public facing external-ip and can be mapped to a domain name with GCP.

http-application-routing addon & nginx-ingress controllers should be used for production use!
