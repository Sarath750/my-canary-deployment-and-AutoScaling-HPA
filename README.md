# my-canary-deployment-and-AutoScaling-HPA

 Canary Deployment (Newer one)
------------------
It's not advisable to move traffic 100% at once, pod may crash. lets say our application runs with 10 pods, but when we are moving traffic to other deployment(v2) it may have only 2 0r 3 pods at start.So, even for autoscaling also it needs some time to increase the pods to the desirable count. because of all these reasons we use Canary deployment.
eg:- in mobilephone we will release the updates in batches wise using this canary deloyment.

vi v2
-- --
90 10
70 30
50 50
20 80
0 100

when we move 100% percent traffic to v2, we can remove v1.

1st deployment: v1
2nd deployment: v2
service: one service(here this service will manage both deployments with some istio rules)

vi deployment-v1.yml
-------------------
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-v1
  labels:
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
        version: v1
    spec:
      containers:
      - name: springboot
        image: sarath750/springboot-hello:v1
        ports:
        - containerPort: 8080

k apply -f deployment-v1.yml



vi deployment-v2.yml
--------------------
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-v2
  labels:
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
        version: v2
    spec:
      containers:
      - name: springboot
        image: sarath750/springboot-hello:v2
        ports:
        - containerPort: 8080

k apply -f deployment-v2.yml

vi service.yml
--------------
---
apiVersion: v1
kind: Service
metadata:
  name: springboot
  labels:
    app: springboot
spec:
  type: LoadBalancer
  selector:
    app: springboot
  ports:
  - name: http
    port: 8080

k apply -f service.yml


vi istio-rules.yml
------------------
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: springboot-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: springboot
spec:
  hosts:
  - "*"
  gateways:
  - springboot-gateway
  http:
  - route:
    - destination:
        host: springboot
        subset: v1
      weight: 80
    - destination:
        host: springboot
        subset: v2
      weight: 20
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: springboot
spec:
  host: springboot
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2


k apply -f istio-rules.yml

Downlaoding Istio
-----------------
echo 'export ISTIO_VERSION="1.5.2"' >> ${HOME}/.bash_profile
source ${HOME}/.bash_profile
mkdir environment
cd ~/environment
curl -L https://istio.io/downloadIstio | sh -

cd ${HOME}/environment/istio-${ISTIO_VERSION}
sudo cp -v bin/istioctl /usr/bin/

istioctl version --remote=false


Installing istio
-----------------
istioctl manifest apply --set profile=demo

kubectl -n istio-system get svc

kubectl -n istio-system get pods



k delete -f .





AutoScaling
-----------
HPA(Horizontal Pod AutoScaling)

git clone <kubernetes URL>
cd /root/kubernetes/AutoScaling/hpa/metrics-server


k apply -f metrics-server


vi deployment.yml
-----------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-deployment
  labels:
    app: springboot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
    spec:
      containers:
      - name: springboot-deployment
        image: sarath750/springboot-hello:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 100m
          requests:
            cpu: 100m

k apply -f deployment.yml


vi service.yml
--------------
---
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
spec:
  type: LoadBalancer
  selector:
    app: springboot
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080

k apply -f service.yml


vi hpa-new.yml
--------------
---
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: springboot-hpa
spec:
  maxReplicas: 5
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: springboot-deployment
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 40
  behavior:
    scaleDown:
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
      - type: Percent
        value: 10
        periodSeconds: 60
      selectPolicy: Min
      stabilizationWindowSeconds: 300
    scaleUp:
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
      - type: Percent
        value: 40
        periodSeconds: 60
      selectPolicy: Max
      stabilizationWindowSeconds: 0

k apply -f hpa-new.yml

k get hpa

k get pods

k exec -it <pod-name> /bin/bash

top

for i in 1 2 3 4; do while : ; do : ; done & done
Duplicate the server
alias k=kubectl
k get all -A
k get pods    
(Now all 5 pods are in running state)


for i in 1 2 3 4; do kill %$i; done
top
k get pods --watch
(Now the pods will be decreased one by one)

