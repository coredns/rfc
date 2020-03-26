# UDP(DNS) Health Check for Kubernetes Deployment

| Status        |  Proposed        |
:-------------- |:---------------------------------------------------- |
| **RFC #**     | [#3](https://github.com/coredns/rfc/pull/3) |
| **Author(s)** | Jayesh Sharma (@[wjayesh](https://github.com/wjayesh))  |
| **Sponsor**   |    |
| **Updated**   | 2020-03-03                                           |
| **Obsoletes** |  |

## Objective

CoreDNS is the cluster DNS server for Kubernetes and is very critical for the overall health of the Kubernetes cluster. It is important to monitor the health of CoreDNS itself and restarting or repairing any CoreDNS pods that are not behaving correctly.  
 
There is a need for a reliable way to validate that the CoreDNS pods are providing DNS resolution services to entities outside of a specific cluster. This health check should be external to the CoreDNS binary and not get affected by CoreDNS outages.
 
## Motivation and Scope


The goal of this project is listed below:

- An application that performs UDP-based DNS requests against the Kubernetes cluster and validates that the received DNS responses match expected responses.
- The application will restart CoreDNS pods via the Kubernetes API when the previously executed validations fail. The exact threshold for triggering this will be tunable and be available as a configurable option. It can be a percentage of failures or some other metric which I will determine over the course of the project. 


While CoreDNS exposes a health check itself in the form of Kubernetes’ `livenessProbe`: 


- The health check is not UDP (DNS) based. There have been cases where the health port is accessible (for TCP) but CoreDNS itself isn't (UDP). This protocol difference means that CoreDNS is unhealthy from a cluster standpoint, but the control plane can't see this.  
- The existing health check is also launched locally (the `kubelet` uses the `livenessProbe`) and the situation could be different for pods accessing it remotely.   

The tool is aimed at the operations team of organisations with multi-node Kubernetes clusters. The tool will require access to the cluster and directly or indirectly manages resources of the clients and as such people operating the binary will need to have those privileges. Manual intervention might be necessary in the case the pods fail repeatedly after restarting and the team needs to be equipped to make changes to the resources allocated to the pods.
 
### Deliverables  


- The deliverable is a standalone Golang binary: 
  - The binary will be a **flexible** application because it could choose its approach to the problem depending on the environment it is deployed in and the access that has been provided to it.   
  - The binary can be deployed anywhere, inside or outside of a Kubernetes cluster. 
  - This binary only concerns itself with the health of CoreDNS deployed on Kubernetes. It doesn’t cover other scenarios. 


- Proper documentation for the product explaining steps and the concepts used. There will be `man` pages for the deployment strategies and for interpreting the response of the binary. The repository will have a `README.md` with important details about the project and scope of future development.
 
 

 
## Design Proposal 

 
The binary is designed keeping in mind the breadth of deployment options available and is made to be flexible to work in all scenarios.
Minimal assumptions about the users’ Kubernetes deployment are made and the primary **focus** is to build a program that could easily be adopted by more teams (which may have different preferences of deployment).


First, the application extracts information about Kubernetes cluster, including its services and pods.
The application does it by querying the cluster via Kubernetes API. If the application is unable to reach the cluster, then a prompt to check the deployment settings is displayed.
     
 
Second, it discovers relevant IP addresses, e.g. cluster and pod IP addresses, of
CoreDNS service and pods in the cluster from the information
received from Kubernetes API.  
The previously discovered cluster IP and the pod IP, both need to be pinged because in real production systems, any point of the system could fail. We cannot make the assumption that Kubernetes networking (forwarding requests from the Service IP to the Pods) will work all the time.  
Apart from this, it’ll be useful to know which Pods are sick, so they can be remedied individually.

If there are no CoreDNS pods or service, then .....TODO
 
 
 
Finally, it iterates through the IP addresses and perform dns lookup on them. The CoreDNS pods are restarted through the API server if pods are up and running but DNS reply does not match the expectation as mentioned in the Objective. 


It is necessary to consider all possible cases because the binary is designed to be flexible to the **choice**  of the user.  
 


## Implementation


I have listed out four possible scenarios that the binary might be deployed in. Out of these, three have a determinate solution and can be implemented right away. The last one will require some hacking and also relies on a feature which is under development.

There are steps which are shared by all cases. 
The Go [client](https://github.com/kubernetes/client-go) for Kubernetes has packages that can discover and authenticate with the API server. The client can also help with the discovery of pods and services. 

The DNS [library](https://github.com/miekg/dns) for Go is used to "dig" the cluster IP and pod IPs through DNS. The output of the command execution will help determine whether the DNS service is healthy or not. 
If the IP of the service changes between the time the cluster-info is pulled and the time of the DNS request, then the best option would be to have an exception handler that will recheck the info if the current query fails (due to wrong IPs).

The restarting of the pods can also be accomplished using the Golang client which talks to the Kubernetes API server. 

### Kubernetes Pod

  Here, the team decides to launch the binary as a health check pod using `kubectl` and a YAML configuration. The steps listed above can then be directly executed with no additional configuration or modifications required.  
 
| **Pros** | **Cons** |
| --- | --- |
| Faster access to the CoreDNS pods/service as all communication within cluster | Consumes resources of the users' cluster for creating pods |
|Easier to debug since less number of point of failures| |


### External App with Pod Management Privileges
![CoreDNSPod](https://user-images.githubusercontent.com/37150991/75658062-9220f900-5c8d-11ea-8c13-f48fbcecc37a.jpeg)
 
 
  Here, the binary itself lives as an external program but will dynamically create a lightweight pod in the cluster (through communication with the API server). The only purpose of this pod inside the cluster would be to execute the dig command (and maybe other debug commands) that the external Golang application sends to this pod. 
  
  We can also get rid of the pod right after a check and create one the next time we are running a check so the main application doesn't actually live on the cluster and isn't affected by it. 


  The pod is just created so that accessing the private k8s cluster network becomes simple (no routing or proxy required). It is straightforward to run a shell in a container (through kubectl exec) and then equally simple to run the dig commands in the shell. 

| **Pros** | **Cons** |
| --- | --- |
| Maintains fast access to cluster as lookup performed within cluster network  | Communication between the binary and the pod in the cluster introduces a point of failure/delay |
| Takes lesser resources compared to first case as the binary lives outside the cluster | Still consumes some memory/cpu of the users' cluster |

### Binary Deployed Externally and DNS Port Exposed
![CoreDNSNodePort](https://user-images.githubusercontent.com/37150991/75658133-b5e43f00-5c8d-11ea-87de-2dc14f965cd3.jpeg)

  Here, the DNS port or the UDP port 53 (in standard deployment) of the CoreDNS deployment is exposed through a service to the external world. The binary, deployed externally, can then use this service to ping the endpoints using the dig command.  
 
| **Pros** | **Cons** |
| --- | --- |
| No cluster resources used | Communication inside the cluster between service and pods is not foolproof and might introduce failures|
| | Service IPs may change between requests and delay is introduced to get the right IPs again |
| | Difficult or impossible to separately ping pod IPs |
### Creation of Pods Forbidden and DNS Port Not Exposed


  In such a case, the binary (living as an external program) would have very limited options to interact with the cluster. However, there is a way this scenario can be tackled elegantly and that is through a port-forward.  


  Port-forward creates a connection between a port on our application host and the port we want to connect to on a pod, inside the cluster.  Although port-forward works only for TCP connections currently, there is an active issue to bring the functionality to UDP ports and I’ll be tracking the issue over the next months to see if something can be hacked together.  


| **Pros** | **Cons** |
| --- | --- |
| No cluster resources used | port-forward not available for UDP, as of now | 
| Fast access as port on the pod directly exposed | |
| Lesser number of point of failures and easier to debug | |

Preliminary API Spec 
-

```Go
type Client struct {
    clientset Clientset
    deploymentOption string
    // more attributes
}
```


```Go
func (c Client) connectToApiServer() { 
/*  
Establish connection with the API server of the cluster   
and authenticate client  
*/
} 
```

```Go
func (c Client) findIPs(caseNumber string) map[string]string { 
/*  
Finds and returns the IPs to be pinged by the `dig` method.  
Depending on the case, either the ClusterIP or the `NodePort`/`LoadBalancer` IP    
is returned.	 
*/
 
// Args: 
// caseNumber: Depending on the case, either the ClusterIP or the `NodePort`/`LoadBalancer` IP  
// is returned. 
// Returns: 
// Map of IPs (and their tags) to be pinged 
 } 
```

```Go
func (c Client) digIP(IPAddress string) []bool { 
/*  
Sends DNSqueries to the input IP address.  
Implements logic to determine if CoreDNS is healthy or not based on 
the output received	
*/ 
 
// Args: 
// IPAddr: Set of addresses to send `dig` requests to.  
// Returns: 
//  A set of true/false values for the input resources 
//  false indicating bad health 
} 
```

```Go
func (c Client) restartCoreDNS(indicators []bool) string { 
/*  
Figure out what resources to repair. Communicate with the API server to restart 
the sick resource  and maintain a healthy state.	
*/ 
 
// Args: 
// indicators: A set of true/false values for the input resources 
//  false indicating bad health 
// Returns: 
//  Status report indicating if operations successful 
} 
```

```Go
func (c Client) deployAndExposePod() []string { 
/*  
Method to create lightweight pod inside the cluster with purpose of 
accessing the Kubernetes networking and running `dig` on the ClusterIP.	
*/ 
 
// Args: 
// None 
 
// Returns: 
//  IP address of the service, pod name and other information 
} 
```
 

Timeline 
-

**Before 27 April:** 

I’ll get comfortable with the Go language and DNS concepts. 


**27 April to 18 May:** 
 
I’ll bond with the community and understand what test systems, coding practices and principles are followed. I’ll stay active on community channels and discuss the idea and solution further to finalize and perfect the approach. 


**18 May to 31st May:** 
 
Develop code to enable the application to determine what scenario and environment it is deployed in.  
I will implement a priority order for the different approaches when two or more are existing simultaneously; after deliberations with the mentors.  
Ultimately, the app at this stage should be able to transfer control to one of the four cases. 
 
**1st June to 7th June:** 
 
Test the existing code rigorously and document the process. 
 
**7th June to 21st June:** 
 
Take on the first case (deployed as a pod) and implement part of the specified API that caters to this situation. A lot of the code written in this period could be reused with some additions or logic control for the remaining cases.  
 
**22nd June to 30th June:** 
 
Test and document the existing code thoroughly.  
 
 
 
**1st July to 7th July:** 
 
Implement the part of the API specific to the second case. Some code will need to be added to get the `NodePort`/`LoadBalancer` IP in this case. Will also look for other modifications required.  
 
**7th July to 14th July:** 
 
Test and document the application up to this stage and ensure the two cases report correct results and function as expected.  
 
**14th July to 21st July:** 
 
Implement the deployAndExposePod function and check compatibility with code in the existing functions. Figure out if any modifications or utilities needed. 
 
**21st July to 28th July:** 
 
Test and document the code and ensure all the three cases work as expected.  
 
**28th July to 10th August:** 
 
Explore options for solving the fourth case and look for any hacks or contributions required to make it work.  
Will also further test the code in different conditions and better the documentation of the entire project. 
 
**10th to 17th August:** 
Buffer to accommodate for unexpected delays or roadblocks.    
 
 

 

## Questions and Discussion Topics

- Are there any deployment cases that I have missed?
- Are there any modifications to the strategies listed above, that should be made?
 
 

