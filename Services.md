
https://docs.docker.com/engine/swarm/services/
(WIP: Need to add the details before this)
Publish ports
	When you create a swarm service, 
		you can publish that service’s ports to hosts outside the swarm in TWO ways:

	1. Rely on the routing mesh. 
		When you publish a service port (Similar to Node port in k8s), 
			Swarm makes the service accessible at the target port on every node 
			regardless of whether there is a task for the service running on that node or not. 
			This is less complex and is the right choice for many types of services.
				docker service create --name my_web \
                        --replicas 3 \
                        --publish published=8080,target=80 \
                        nginx


	2. Publish a service task’s port directly on the swarm node where that service is running. 
		Bypasses the routing mesh 
		Provides the maximum flexibility
			ability for you to develop your own routing framework. 
		You need to keeping track of where each task is running 
		routing requests to the tasks, and load-balancing across the nodes.
			docker service create \
			  --mode global \
			  --publish mode=host,target=80,published=8080 \
			  --name=nginx \
			  nginx:latest


Publish a service’s ports using the routing mesh
	To publish a service’s ports externally to the swarm
		use the --publish <PUBLISHED-PORT>:<SERVICE-PORT> flag. 
		The swarm makes the service accessible at the published port on every swarm node. If an external host connects to that port on any swarm node, the routing mesh routes it to a task. The external host does not need to know the IP addresses or internally-used ports of the service tasks to interact with the service. When a user or process connects to a service, any worker node running a service task may respond. For more details about swarm service networking, see Manage swarm service networks.

Example: Run a three-task Nginx service on 10-node swarm
--------------------------------------------------------
Number of nodes: 10-node swarm
deploy 3 instances Nginx service:

$ docker service create --name my_web \
                        --replicas 3 \
                        --publish published=8080,target=80 \
                        nginx
						
Three tasks run on up to three nodes. 
You don’t need to know which nodes are running the tasks
connecting to port 8080 on any of the 10 nodes connects you to one of the three nginx tasks. 
You can test this using curl. The following example assumes that localhost is one of the swarm nodes. If this is not the case, or localhost does not resolve to an IP address on your host, substitute the host’s IP address or resolvable host name.

The HTML output is truncated:

$ curl localhost:8080

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...truncated...
</html>
Subsequent connections may be routed to the same swarm node or a different one.

Publish a service’s ports directly on the swarm node
---------------------------------------------------
Using the routing mesh may not be the right choice for your application if you need to make routing decisions based on application state or you need total control of the process for routing requests to your service’s tasks. To publish a service’s port directly on the node where it is running, use the mode=host option to the --publish flag.

Note: If you publish a service’s ports directly on the swarm node using mode=host and also set published=<PORT> this creates an implicit limitation that you can only run one task for that service on a given swarm node. You can work around this by specifying published without a port definition, which causes Docker to assign a random port for each task.

In addition, if you use mode=host and you do not use the --mode=global flag on docker service create, it is difficult to know which nodes are running the service to route work to them.

Example: Run an nginx web server service on every swarm node
nginx is an open source reverse proxy, load balancer, HTTP cache, and a web server. If you run nginx as a service using the routing mesh, connecting to the nginx port on any swarm node shows you the web page for (effectively) a random swarm node running the service.

The following example runs nginx as a service on each node in your swarm and exposes nginx port locally on each swarm node.

$ docker service create \
  --mode global \
  --publish mode=host,target=80,published=8080 \
  --name=nginx \
  nginx:latest
You can reach the nginx server on port 8080 of every swarm node. If you add a node to the swarm, a nginx task is started on it. You cannot start another service or container on any swarm node which binds to port 8080.

Note: This is a naive example. Creating an application-layer routing framework for a multi-tiered service is complex and out of scope for this topic.

-------------------------------------------------------

Grant a service access to secrets
	--secret
	
Customize a service’s isolation mode
	Applies to only windows - refer https://docs.docker.com/engine/swarm/services/
	
Control service placement
-------------------------
	There are various ways for you to control 
		scale and 
		placement 
			of services on different nodes.
1. Can specify 
	whether the service needs to run a specific number of replicas 
		Replicated service
	or 
	should run globally on every worker node. 
		Gloabl service

2. Can configure the service’s 
	CPU or memory requirements
	service only runs on nodes which can meet those requirements.

3. Placement constraints 
	Configure the service to run only on nodes with specific (arbitrary) metadata set
	Deployment will fail if appropriate nodes do not exist. 
	For e.g., 
		can specify:service should only run on nodes 
			label pci_compliant = true.
			
	docker service create --replicas 2 --name webserver --constraint 'node.role==manager' nginx
		will get created on the manage only.
		
	or node.hostname == <hostname>

	docker node update --label-add mongo=primary <node name>
	docker service create --name webserver --publish 80:80 --constraint 'node.labels.mongo == primary' nginx
	
	
4. Placement preferences 
	Can apply an arbitrary label with a range of values to each node
	spread your service’s tasks across those nodes using an algorithm. 
	Currently: only supported algorithm is spread
		tries to place them evenly. 
		
		For e.g., 
		
		If you apply the following labels to the nodes:
			node-1: datacenter=us-east, disk=ssd
			node-2: datacenter=us-east, disk=ssd
			node-3: datacenter=us-west, disk=sas
			node-4: datacenter=us-west, disk=nl-sas
			
			docker node update --label-add datacenter=us-east --label-add disk=ssd node-1
			docker node update --label-add datacenter=us-east --label-add disk=ssd node-2
			docker node update --label-add datacenter=us-west --label-add disk=ssd node-3
			docker node update --label-add datacenter=us-west --label-add disk=nl-sas node-4

			docker service create --replicas 2 --name webserver --placement-pref 'spread=node.labels.datacenter' nginx


		Docker Swarm would 
			try to "spread" the 2 replicas across both datacenters
			Mostly 
				1 replica on node-1/node-2
				and the other replica on node-3/node-4.
		Why Mostly?
			placement constraints, 
			placement preferences and 
			other node-specific limitations also would be considered while assinging.
		Node-specific limitations - limitations imposed by us
			CPU limitation
			memory limitation (Service should run on nodes with x cpu/memory)
			demoted nodes ect.


Unlike constraints
	placement preferences are best-effort
	service does not fail to deploy if no nodes can satisfy the preference. 
	If you specify a placement preference for a service
		nodes that match that preference are ranked higher 
			when the swarm managers decide which nodes should run the service tasks. 
