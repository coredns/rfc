# Title of RFC

| Status        | Proposed       |
:-------------- |:---------------------------------------------------- |
| **RFC #**     | [#9](https://github.com/coredns/rfc/pull/9) |
| **Author(s)** | Dhiraj Sharma (@Dhiraj240) |
| **Sponsor**   | Yong Tang (@yongtang), Paul Greenberg (@greenpau)   |
| **Updated**   | 2020-05-16                                           |
| **Obsoletes** |  |

## Objective

The project aims at monitoring the health of CoreDNS itself and restarting or repair of CoreDNS pods that are not behaving correctly. Moreover an external health checkup via UDP will be used and using golang CoreDNS pods will restart by interacting with kubernets API.

## Motivation and Scope

This is a valuable project for checking the health of CoreDNS. It will be useful in repairing the CoreDNS pods where an engineer has to manually check up until a GET request is exposed in health plugin.

The use of external UDP(DNS) health checkup based system will be useful for CoreDNS by interacting with Kubernets API.


If CoreDNS crashes we can't check this up. It is not good in cluster point of view as Kubernets can't check UDP health.The deliverable of this project is a golang program that could be deployed in a Kubernetes cluster independently while at the same time, monitoring CoreDNS pods in the same cluster and interacting Kubernetes API (server) to restart CoreDNS pods as needed.

## Design Proposal

**Check CoreDNS health through UDP(DNS) externally**

1. As of now the health plugin in CoreDNS waits for a GET request from a port and currently the design is not UDP(DNS) based. I read about the [plugins](https://coredns.io/manual/plugins/) which provided me vitaal information on building the system on how to communicate.

2. I somewhat studied this [health plugin](https://github.com/coredns/coredns/tree/master/plugin/health) and understood its code, now an externally system would be developed using the [grpc](https://kubernetes.io/blog/2018/10/01/health-checking-grpc-servers-on-kubernetes/) where the [go package is available for it](https://godoc.org/github.com/grpc/grpc-go/health).Now here I will be needing help from mentors on how can I adjust servers.

3. Below is the demo for UDP(DNS) health check.

A simple [DNS process](https://miro.medium.com/max/1400/1*qQkVbFQfBSD7SX48jfoT-w.png).As far as I read CoreDNS documentation I know we have multiple servers so here for DNS we can use CNAME. Like we have several subdomains for corp server and we could call subdomains from main corp.Like wise this [architecture](https://miro.medium.com/max/1292/0*RB3WScUb-45OSGQH).

To create a DNS server we have to listen to a port.
By default it will run on port 53.

//Listen on UDP Port
	addr := net.UDPAddr{
		Port: 8090,
		IP:   net.ParseIP("127.0.0.1"),
	}
	u, _ := net.ListenUDP("udp", &addr)

I used a Google Packet to listen to a port 

// Wait to get request on that port
	for {
		tmp := make([]byte, 1024)
		_, addr, _ := u.ReadFrom(tmp)
		clientAddr := addr
		packet := gopacket.NewPacket(tmp, layers.LayerTypeDNS, gopacket.Default)
		dnsPacket := packet.Layer(layers.LayerTypeDNS)
		tcp, _ := dnsPacket.(*layers.DNS)
		serveDNS(u, clientAddr, tcp)
	} 

To build DNS server in Go there is small prototype

package main

import (
	"fmt"
	"net"

	"github.com/google/gopacket"
	layers "github.com/google/gopacket/layers"
	"github.com/grpc/grpc-go/health"
)	

var records map[string]string

func main() {
	records = map[string]string{
		"google.com": "216.58.196.142",
		"amazon.com": "176.32.103.205",
	}

	//Listen on UDP Port
	addr := net.UDPAddr{
		Port: 8090,
		IP:   net.ParseIP("127.0.0.1"),
	}
	u, _ := net.ListenUDP("udp", &addr)

	// Wait to get request on that port
	for {
		tmp := make([]byte, 1024)
		_, addr, _ := u.ReadFrom(tmp)
		clientAddr := addr
		packet := gopacket.NewPacket(tmp, layers.LayerTypeDNS, gopacket.Default)
		dnsPacket := packet.Layer(layers.LayerTypeDNS)
		tcp, _ := dnsPacket.(*layers.DNS)
		serveDNS(u, clientAddr, tcp)
	}
}

func serveDNS(u *net.UDPConn, clientAddr net.Addr, request *layers.DNS) {
	replyMess := request
	var dnsAnswer layers.DNSResourceRecord
	dnsAnswer.Type = layers.DNSTypeA
	var ip string
	var err error
	var ok bool
	ip, ok = records[string(request.Questions[0].Name)]
	if !ok {
		//Todo: Log no data present for the IP and handle:todo
	}
	a, _, _ := net.ParseCIDR(ip + "/24")
	dnsAnswer.Type = layers.DNSTypeA
	dnsAnswer.IP = a
	dnsAnswer.Name = []byte(request.Questions[0].Name)
	fmt.Println(request.Questions[0].Name)
	dnsAnswer.Class = layers.DNSClassIN
	replyMess.QR = true
	replyMess.ANCount = 1
	replyMess.OpCode = layers.DNSOpCodeNotify
	replyMess.AA = true
	replyMess.Answers = append(replyMess.Answers, dnsAnswer)
	replyMess.ResponseCode = layers.DNSResponseCodeNoErr
	buf := gopacket.NewSerializeBuffer()
	opts := gopacket.SerializeOptions{} // See SerializeOptions for more details.
	err = replyMess.SerializeTo(buf, opts)
	if err != nil {
		panic(err)
	}
	u.WriteTo(buf.Bytes(), clientAddr)
}

Above the records are initialized in the begining.
The serveDNS function gets the connection, client Address and the query request as parameters. The request message is then used to mock response message and the values that have to change in response is then changed.
Then the response message is serialized to UDP message format and write back to the client using the client address obtained on receiving the DNS message as UDP package.

**CoreDNS pods repair and restart via Kubernets API**

To restart the pods using Kubernets API, we can follow this [documentation](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/), here too I need to consult with mentors for the right and specific approach.
We then need to make required changes.

I would use kubectl to access the API. But, I just saw that we have other alternative approach too [here](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/)

Using [this unofficial documentation](https://unofficial-kubernetes.readthedocs.io/en/latest/concepts/services-networking/dns-pod-service/) we can also build a workaround.

## Timeline

**Before May 18**
1. Read more documentation of current health system and start digging into to find the IP addresses to build externally setup using UDP(DNS)

**May 20 to June 22**

1. Likewise working biweekly. Firstly work in Go to setup DNS server and listening to ports.Build corresponding tests for that, in order to align with process of development.

2. Use other functionality of grpc package of Go and consult with mentors for use case.

**June 23 to July 27**

1. Work around to use kubectl to access the API and follow up with Kubernets documentation to restart the pod services which fails.

2. Here the system would be tested before the final week to avoid the use cases flaw pointed by mentors.

## Alternatives Considered
* Internal health checkup has several cases which could probably not work in clusters hence following up with the above approach only.

## Questions and Discussion Topics

I need to know on testing too so is it fine that I take sometime to build it part by part, means checking end to end services.
