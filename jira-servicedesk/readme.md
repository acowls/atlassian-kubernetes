# Deploying with Ansible
* .yaml files are ansible, .yml files are k8s manifests
* envs.yaml contains most key settings used in the deployment

# Notes:
This deployment has been updated to use the official [Atlassian Jira Software Images](https://hub.docker.com/r/atlassian/jira-servicedesk). There's a few changes included in these from the prior images. As such, the configmaps are not required and do have some problems with how permissions are managed, but using ENVs & secrets is more secure. 

This is modified from the good work of https://github.com/Bonn93/atlassian-kubernetes

Notes on dcoker from Atlassian: https://hub.docker.com/r/atlassian/jira-servicedesk
Note: the Jira application runs as the user jira, with a UID and GID of 2001. 

The k8s objects have been split into several files, this helps with troubleshooting as it makes identifcation of failures in each file more apparent. 

A example envs.yml file has been provided, this contains nearly all configurable options and tweaks. Feel free to extend this further by using Ansible {{}} jinja format. 

It is intended that an Ingress Controller is used in front to manage sticky sessions and load balance traffic between pods.

# Target Environment is Azure, with:
* Azure Web Application Firewall and Gateway
* Azure DBaaS hosted in Azure with JDBC connection string
* Single node K8
* Azure Managed Disk for storage

# Quick Deploy
```ansible-playbook jsd_deploy_playbook.yaml -e @envs.yaml```

# Quik Deploys Steps
1. To create the storage class
```kubectl apply -f azure-files-pod.yaml```

# Mapping custom files
You may have a custom dbconfig.xml or server.xml requirements. This repo includes basic support to map these files in such as the server.xml for HTTP2 support. Please note that due to how permissions are managed, this may cause start-up failures until bugs are resolved. 

##### HTTP2 Support:
* HTTP2 support has been added as an example in jira_http2_configmap.yml which adds an upgrade connector ```<UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol"/>``` use this as an example to map objects into the container. You could use this to configure log4j properties too. 

# StatefulSets
* This deployment uses [StatefulSets](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/) and is set to 1 replica. You can scale this deployment after the following is completed
* Main DB, App and Licence is configured @ 1 replica
* The jira-servicedesk-0 pod is online and healthy
```kubectl scale sts/jira-servicedesk --replicas=2```

# Scaling Jira and ZDU Upgrades
* ZDU is only for JIRA Data Centre Edition

# Customizing Options
This guide relies on NFS volumes and static node-port type load balancers. These can be changed. The Jira docker container accepts several ENV variables to configure JVM or Tomcat HTTP options, these are documented in the official atlassian jira images as linked above. 

# Kubernetes Specific:

##### Namespace
* Target K8s Namespace. You may have development, test and production namespaces for your services and applications. 

```test```

##### Jiraservicename
* Service label used to gather objects. This is an object used to direct traffic and manage the k8s objects easily.

```jira-servicedesk```

##### Appname
* Name of the application. A friendly name for the application

```jsd-dev```

##### Jiravolumeclaim
* K8s PVC object to claim the volume

```jiranfspvc```

##### Jiravolumename
* Used in Deployment

```jsddata```

##### jirapv
* Jira Persistent Volume used in deployment

```jsddata```

##### Storagesize
* Capacity in GB for the Jira Home. This value must match everywhere! 

```10Gi```

##### nfsserverip
* IP or FQDN for NFS Host

```10.1.1.22```

##### nfspath
* NFS Mount Path ( nfs export )

```/data/jira-prod/```

##### jirak8servicename
* used with k8s service and ingress controllers.

```jira-svc```

##### nodeip1,2
* Used with local load balancers or node ports to expose services outside without ingress controllers
* typically takes the node-port of each node in the k8s cluster
* Production services should use Ingress controllers with session affinity support

```10.1.1.25```

# Jira/Tomcat/JVM
##### jvm
* Xms/Xmx value. Recommended to set the name. Not too big, not too small.

```2048m```

##### Topcat properties
* tomcat properties are set with the following values in the envs.yaml

```PROXYNAME```


```PROXYSCHEME```


```PROXYSECURE```


* Proxy Name is your FQDN in which your reverse proxy/load balancer listens for the service, such as bamboo.company.com
* Proxy Scheme tells tomcat if this is HTTP or HTTPS
* Proxy Secure informs tomcat if SSL is in use and allows for better re-writing and security. This value is hard coded into the deployment.yml due to bugs between ansible and go templates not reading this value correctly. If HTTPS, set this to "true", if HTTP, set this to "false"
