### Custom resources

#### PacketCapture CR

##### Spec

Targeting a pod:
~~~
apiVersion: packetcapture/v1alpha1
kind: PacketCapture
metadata:
  name: pcap1
  namespace: pcapper
spec:
  targetNamespace: test-namespace
  targetPod: test-pod
  captureSeconds: "600"
  captureFilter: "icmp and host 192.168.1.10"
~~~

Targeting an interface:
~~~
apiVersion: packetcapture/v1alpha1
kind: PacketCapture
metadata:
  name: pcap2
  namespace: pcapper
spec:
  targetNode: worker-01
  targetInterface: eno1
  captureSeconds: "600"
  captureFilter: "icmp and host 192.168.1.10"
~~~

##### Status fields

Packet capture CRs will have the following status fields:
~~~
status:
  progress: <scheduled|running|finished>
~~~
