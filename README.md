

### Introduction
This project is a prototype of a Service Function Chaining (SFC) application written in Python for Ryu SDN controller. It uses flat network which can be referenced as SFC enabled domain.
The only function which is performed by Service Functions (which I may refer as Virtual Network Functions) is further traffic forwarding. So those VNFs referenced here as forwarders.   
The application enforces forwarding rules for a particular flow to OpenFlow enabled network  so that traffic  is passed through the defined chain of Service Functions.  

### Terminology mess notice
There are several organisations working on fostering SDN and NFV. The Internet Engineering Task Force, Working Group for service function Chaining (IETF WG SFC) and European Telecommunications Standards Institute, Industry Specification Group for NFV (ETSI ISG NFV) are two to mention.
They tend to call the same things in different wordings.
Where IETF call an intended path a flow traverses as a _Service Function Chain_, ETSI refers the same as a _Forwarding Graph_. The same relates to a _Service Function_ (IETF) and _(Virtual) Network Function_ (IETF) which are an instance of something that processes data flows. 
So those terms picked from both of standardisation bodies and used here interchangeable. Although for VNF it is not that important to emphasise its virtual nature in this project, I widely use acronym _VNF_ here.

### How it works

A Network Function Forwarding Graph defines a sequence of NFs that packets traverse. VNF Forwarding Graph provides the logical connectivity between VNFs.
Controller application reads from the Service Catalogue a description of a service intended for a packet flow generated by a tenant. The service description is provisioned by OSS/BSS or manually in test environment. It supposed to be a list of VNFs. This information along with VNF description is sufficient to formulate and enforce Open Flow rules to the network. Actual graphs achieved through manipulation of address information in packet headers followed by packet forwarding. 

##### VNF self-registration

VNF self-registration is a service function discovery mechanism. 

![self-registration](https://cloud.githubusercontent.com/assets/19608626/20056843/4ec25a54-a4f9-11e6-9689-fa366a200ab4.jpg)


A cattle approach implies a service being instantiated from a resource pool embracing computing, storage, network resources. Virtual infrastructure manager,  is responsible for scheduling and spinning up a virtual machine. In this case the location of a service (we will call it service function) is known with the granularity of the resource pool location, which is not enough for the purpose of Forwarding Graph building. To address this challenge we present self-registration functionality which allows a service announce its presence in the network. An assistant process on the VM emits registration message to the network on behalf of the service. This message contains the descriptive information on what kind of service it is, what the role the emitting interface plays (in, out, in-out), whether the service is bidirectional or asymmetric and so forth. The service itself, nevertheless, doesn’t have all the necessary information.  This is where an Open Flow capable switch steps in.

![self-registration-flow-chart](https://cloud.githubusercontent.com/assets/19608626/20056893/8974477a-a4f9-11e6-8e1a-31aa6b2edd58.jpg)

It wraps the registration packet into a packet_in OpenFlow  message and sends it to the SDN controller. Packet_in message includes network specific information, such as service address locator, which is its mac address, datapath identificator which is a unique switch id, and a port id , through which the  registration message has been received. The controlling application parses packet_in message, retrieves the information from it, decapsulates the registration message, decodes it as well, and passes all the retrieved information to the database.  


#### Rule enforcement

In the setup we have SDN network controlled by SDN controller on top of which a SFC application is running. The application exposes REST API to accept directives from OSS/BSS system. The application is integrated with the database, where information on registered service functions, services and flows is stored. This database represents a Service Catalog. There is no any OSS/BSS system, (so I’m going to play a role of OSS/BSS system), service definitions and flow to service bindings prepared manually as well as sending request to the REST APIs of the application.

![sfc-flow-chart](https://cloud.githubusercontent.com/assets/19608626/20056916/a022bfc4-a4f9-11e6-9754-8c4427b701d3.jpg)


The process goes like this:  

1. SF discovery: VNF self-registration. Rightmost (see pic.) DB table  is populated with VNF characteristics. 
2. OSS/BSS populates DB tables describing service and Flow to service binding, where flow specification is included.
3. OSS/BSS requests to start SFC for a flow. Flow is referenced by ID.
4. SFC application interrogates DB on the flow specification...
5. ... and installs the catching rule on the network. It is assumed that the ingress point of the flow is not known. 
6. As soon the traffic of interest appears in SFC-enabled domain, the event reported to SFC application. At this moment the ingress point is revealed.
7. Catching rule is removed and steering rule is installed along the path of service chain.
8. Traffic is steered. 


The application can be improved by adding support of the following functionalities:

1. Interface type support. Currently all the interfaces are treated as inout type. Refining the requests to the Service Catalog Data Base will allow to implement multi-homed service function with defined flow direction (ex. Firewall with inside and outside interfaces). **[DONE]**
2. Symmetric flow support can be a convenient feature to add reverse forwarding graph through symmetric service functions automagically. Symmetric or bidirectional functions are those which require traffic to be passed in both directions: uplink and downlink so that service could be provided (example: NAT). Not having that feature is not critical, but requires explicit definition of the reverse forwarding graph.
3. Group support is required when several instances of a service function are deployed. The absence of it can be worked around by using Load Balancers in front of actual Network Functions. 
6. Etc.: Richer set of protocol fields,  wildcard logic, VNF statuses and other enhancements.

### Demonstration Environment

... to be added

### Demonstration Instructions

The demonstration includes prepopulated flow table in the database, which, of course, can be modified. Flow 3 is used here which specification  describes a flow from h1 (10.0.0.1) to h5 (10.0.0.5)
Script example.py does all the magic of running mininet, interconnecting hosts and running self-registartation on them.

1. Open four terminals
2. Start Ryu application in the 1st terminal:
   * 1st terminal: ``` /home/ubuntu/ryu/bin/ryu-manager --verbose ./sfc_app.py```
3. Start test topology in the 2nd terminal:
   * 2nd terminal: ```sudo ./example.py```
4. Clear flow 3 (the application preinstalls one when get started):
   * 3rd terminal: ``` curl -v http://127.0.0.1:8080/delete_flow/3```
5. Check OpenFlow rules before SFC applied:
   * 2nd terminal: ```mininet> h1 tracepath h5```
   * 4th terminal: ```sudo ovs-ofctl -O OpenFlow13 dump-flows s3 | grep 10.0.0.5```
   * One hop can be seen, no rules are insatlled
6. Apply Service Function Chain and check catching rules being installed on OF switches:
   * 3rd terminal: ```curl -v http://127.0.0.1:8080/add_flow/3```
   * 4th terminal: ```sudo ovs-ofctl -O OpenFlow13 dump-flows s3 | grep 10.0.0.5```
   * After flow application a catching rule is seen on OF switch.
7. Start traffic running, check steering rules:
   * 2nd terminal: ```mininet> h1 tracepath h5```
   * 4th terminal: ```sudo ovs-ofctl -O OpenFlow13 dump-flows s3 | grep 10.0.0.5```
   * Now traffic passes several hops, a catching rule has been replaced with a steering rule.
8. Delete flows
   * 3rd terminal: ```curl -v http://127.0.0.1:8080/delete_flow/3```
   * 2nd terminal: ```mininet> h1 tracepath h5```
   * 4th terminal: ``` sudo ovs-ofctl -O OpenFlow13 dump-flows s3 | grep 10.0.0.5```
   * Data flow passes one hop again, no related rules seen on OF switch
   


