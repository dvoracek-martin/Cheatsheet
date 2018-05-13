##GitHub
Projects meant to be part of the java-project can be found on GitHub under these URLs:

`https://github.com/dvoracek-martin/java-project-gateway`
`https://github.com/dvoracek-martin/time-provider`

#java-project-gateway
A gateway and a load balancer. Is listening on `port 80`

### deployment.yml

```
apiVersion: apps/v1
                            kind: Deployment
                            metadata:
                              name: java-project-gateway
                            spec:
                              selector:
                                matchLabels:
                                  run: java-project-gateway
                              replicas: 1
                              template:
                                metadata:
                                  labels:
                                    run: java-project-gateway
                                spec:
                                  containers:
                                  - name: java-project-gateway
                                    image: dworza/java-project-gateway:latest
                                    imagePullPolicy: "Always"
                                    ports:
                                    - containerPort: 80
```

###Useful commands


Expose java-project-gateway to be accessible outside of the cluster:

`kubectl expose Deployment java-project-gateway --port=80 --target-port=80 --name=load-balancer --external-ip=46.28.110.65 --type=LoadBalancer`

Autoscale java-project-gateway to run in at least 2 and max 5 pods, depending on the CPU load:
 
`kubectl hpa deployment java-project-gateway --min=2 --max=5 --cpu-percent=50`

##time-provider
Service for showing the exact time. Internally is listening on the `port 90`. Though it's not accessible directly from the internet and has to be accessed through the gateway only, under the URL:[http://java-project.com/api/time/v1](http://java-project.com/api/time/v1 )
### deployment.yml

```
apiVersion: v1
kind: Service
metadata:
  name: java-project
spec:
  selector:
    name: time-provider-DNS-service
  clusterIP: None
  ports:
  - name: time-provider-port
    port: 90
    targetPort: 90

---

apiVersion: v1
kind: Pod
metadata:
  name: time-provider
  labels:
    name: time-provider
spec:
  hostname: time-provider
  subdomain: default-subdomain
  containers:
  - image:  dworza/time-provider:latest
    command:
    name: time-provider

```

##ELK STACK

In this case it's not a real ELK stack, because Logstash was slowing down the performance way too much. Because of this the data harvested 
by Filebeat are shipped directly to the Elasticsearch. Filebeat is for easier configuration running as a service, whereas 
Elasticsearch and Kibana are running in a docker container. Kibana can be accessed at URL [java-project.com:5601](http://java-project.com:8080 ).

###Useful commands
`service filebeat start`

`docker run -d -p 9200:9200 -p 9300:9300 -it -h elasticsearch --name elasticsearch elasticsearch`

`docker run -d -p 9200:9200 -p 9300:9300 -it -h elasticsearch --name elasticsearch elasticsearch`

The `/etc/filebeat/filebeat.yml` config file is as follows:
```
filebeat.prospectors:
    -
      paths:
        - "/var/log/containers/java-project-gateway*.log"
        - "/var/log/containers/time-provider*.log"
      fields: {log_type: containers}
      ignore_older: 5m
      symlinks: true
      json.message_key: log
      json.keys_under_root: true
      json.add_error_key: true
      multiline.pattern: '^\d{4}-\d{2}-\d{2}'
      multiline.match: after
      multiline.negate: true
      document_type: kube-logs



output.elasticsearch:
  hosts: ["localhost:9200"]

```

##Jenkins 
Jenkins is running as a service as well and is accessible on URL [java-project.com:8080](http://java-project.com:8080 ).
Whenever a branch of any of the main projects is merged to the master branch, the jenkins pipeline will build the project,
push a new docker image to the given repository in [https://hub.docker.com/r/dworza](https://hub.docker.com/r/dworza ) and 
update the deployment file under `/var/lib/jenkins/workspace/{project_workspace}/`.  Redeployment to Kubernetes is not implemented yet,
therefore one needs to get to  `/var/lib/jenkins/workspace/{project_workspace}/` and run `kubectl apply -f deployment.yml`.

The jenkins pipeline is set to work with Spring-Boot projects, so they're built by maven before the docker image is created.
Sample Dockerfile:

```
FROM openjdk
ADD target/gateway.jar gateway.jar
ENTRYPOINT ["java", "-jar", "/gateway.jar"]
EXPOSE 80
``` 