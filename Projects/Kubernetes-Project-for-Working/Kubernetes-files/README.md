## ÇÖZÜM

1: 5 node'lu "1 Master + 4 Worker Node" bir Kubernetes cluster kurun.
```
minikube start --node=5
```
2: "test" ve "production" adlarında 2 namespace oluşturun.
```
kubectl apply -f .\namespaces.yaml
```
3: "junior" isimli grubun "test" namespace'inde tüm kaynaklar üstünde "okuma, listeleme, yaratma..." gibi tüm haklara, "production" namespace'inde ise tüm kaynaklar üstünde sadece "okuma ve listeleme" haklarına sahip olacağı rol oluşturup bunları grupla ilişkilendirin. Aynı şekilde "senior" isimli grubun "production" ve "test" namespace'lerindeki tüm kaynaklar üstünde "okuma, listeleme, yaratma..." gibi tüm haklara ve cluster üstündeki diğer kaynaklar üstünde ise sadece "okuma ve listeleme" haklarına sahip olacağı rol oluşturup bunları grupla ilişkilendirin.
```
kubectl apply -f .\junior-production-rolebinding.yaml

kubectl apply -f .\junior-test-rolebinding.yml

kubectl apply -f .\senior-cluster-rolebinding.yaml

kubectl apply -f .\senior-production-rolebinding.yaml

kubectl apply -f .\senior-test-rolebinding.yaml
```
4: Sizin seçeceğiniz bir "ingress controller" kurun.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/cloud/deploy.yaml
minikube addons enable ingress
```
5: Cluster'da seçeceğiniz 3 worker node sadece "production" ortamında deploy edeceğiniz ve cluster tarafından oluşturulan podlar schedule edilebilsin. Bunun dışındaki podların bu worker node üstünde oluşturulmamasını sağlayın.
```
node1=$(kubectl get no -o jsonpath="{.items[1].metadata.name}")

node2=$(kubectl get no -o jsonpath="{.items[2].metadata.name}")

node3=$(kubectl get no -o jsonpath="{.items[3].metadata.name}")

kubectl taint node $node1 tier=production:NoSchedule

kubectl taint node $node2 tier=production:NoSchedule

kubectl taint node $node3 tier=production:NoSchedule

kubectl label node $node1 tier=production

kubectl label node $node2 tier=production

kubectl label node $node3 tier=production
```
6: Wordpress uygulamasını hem "test" hem de "production" namespace'lerinde deploy edin. (wordpress:latest ve mysql:5.6 imajlarından oluşturulacak)
```
kubeectl apply -f .\wordpress-mysql-deployment-production.yaml
kubeectl apply -f .\wordpress-mysql-deployment-test.yaml
```
7: "test" namespace'inde deploy edilen wordpress uygulaması "testblog.example.com", "production" namespace'inde deploy edilen wordpress uygulaması "companyblog.example.com" olarak ingress üstünden dış dünyaya expose edilecek.
```
kubeectl apply -f .\ingress.yaml
```
8: "production" namespace'inde "nginx" imajından, 5 replikalı, update stratejisi olarak aynı anda 2 pod'un update edilebileceği bir deployment oluşturun. "/healthcheck" endpoint'ini sorgulayan "liveness probe" ve "/ready" endpoint'ini sorgulayan bir "readiness probe" tanımları da olsun.
```
kubeectl apply -f .\deployment.yaml
```
9: Bir önceki görevde oluşturduğunuz deployment'i "loadbalancer" tipi bir servisle dış dünyadan erişilebilir hale getirin.
```
kubectl expose deployment k8s-deployment --type=LoadBalancer -n production
```
10: Tüm Probe ları yorum satırına alın ve çalıştığını kontrol edin.

11: "fluentd" uygulamasını bir "daemonset" olarak cluster'da deploy edin.
```
kubeectl apply -f .\deamonset.yaml
```
12: 2 node'lu bir "mongodb" cluster'i "statefulset" olarak cluster'da deploy edin. "mongodb" cluster'ın çalıştığından emin olun.

```
$ kubectl apply -f ./yaml/statefulset.yaml

$ kubectl exec -it mongostatefulset-0 -- bash

root@mongostatefulset-0:/# mongo

> rs.initiate({ _id: "MainRepSet", version: 1, 
members: [ 
 { _id: 0, host: "mongostatefulset-0.mongo-svc.default.svc.cluster.local:27017" }, 
 { _id: 1, host: "mongostatefulset-1.mongo-svc.default.svc.cluster.local:27017" } ]});

MainRepSet:PRIMARY> db.getSiblingDB("admin").createUser({
...       user : "mongoadmin",
...       pwd  : "P@ssw0rd!1",
...       roles: [ { role: "root", db: "admin" } ]
...  });

MainRepSet:PRIMARY> rs.status();
```
13: Cluster'da tüm objeler üstünde "okuma ve listeleme" haklarına sahip bir "service account" oluşturun. Bu service account’u bağladığınız bir pod oluşturun ve bağlanarak "curl" ile cluster’daki tüm podları listeleyin.

```
$ kubectl apply -f ./yaml/serviceaccount.yaml

$ kubectl exec -it pod-proje -- bash

bash-5.0# CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

bash-5.0# TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

bash-5.0# curl --cacert $CERT https://kubernetes/api/v1/pods --header "Authorization:Bearer $TOKEN" | jq '.items[].metadata.name'
```
14: Worker node'lardan bir tanesinin üzerindeki tüm podları tahliye edin ve ardından yeni pod schedule edilememesini sağlayın.
```
$ nodedrain=$(kubectl get no -o jsonpath="{.items[3].metadata.name}")

$ kubectl drain $nodedrain --ignore-daemonsets --delete-local-data

$ kubectl cordon $nodedrain

```











