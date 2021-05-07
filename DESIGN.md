### Implementation details

#### Operator

The operator will be deployed in namespace `pcapper` and will only be able to manipulate objects inside this namespace.

The operator pod is responsible for deploying a single DaemonSet called `pcapper`.

The operator is responsible for deploying the `PacketCapture` CRD.

#### DaemonSet pcapper

The DaemonSet will contain a toleration for all nodes and will thus spawn a pod on every single node. 

Pods will run with host networking and with the necessary CAPs to run a packet capture.

The DaemonSet will expose a given pod's nodeName with:
~~~
spec:
  containers:
    - env:
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
      - name: SCAN_INTERVAL
        value: "30"
      - name: MAX_TOTAL_FILESIZE
        value: "10G"
      - name: MIN_FREE_DISK
        value: "20G"
~~~

The pods will run a golang binary called `pcapper`. 

#### binary pcapper

The `pcapper` binary will call into the API every `SCAN_INTERVAL` seconds to check if a new `PacketCapture` CR was created. The `pcapper` binary is aware of its current hostname through the `NODE_NAME` environment variable.

A new `PacketCapture` CR does not have a `progress` status field, and no `node` is set. 

If a given `targetNamespace`/`targetPod` or `targetNode`/`targetInterface` combination matches the the `pcapper` pod's host, then the `pcapper` pod will annotate the `PacketCapture` CR with `node: <node name>` and `progress: scheduled`. 
All `pcapper` binaries will still investigate this `PacketCapture` for as long as it is marked as `scheduled` and the node name matches this pod's node name. As soon as a `PacketCapture` passes into any other state such as `running`, it will be completely ignored. 

The current binary will now investigate `MAX_TOTAL_FILESIZE` and `MIN_FREE_DISK` on the hostmount with `du -sh /hostmount` and `df -h /hostmount`. If these constraints are not met, it will set `progress` to `failure` and generate an according event.

Otherwise, the `pcapper` binary will find the correct interface to run a tcpdump. It will run `timeout $(captureSeconds) tcpdump -w /hostmount/${NODE_NAME}.${PacketCapture.metadata.name}.$(date +%s).pcap -i ${INTERFACE_NAME} -C ${maxFileSize} -W ${fileCount} -Z root`. 

After the packet capture finishes, the `pcapper` binary will update the CR's status to `finished` and generate an appropriate event.

### Custom resources

#### PacketCapture CRD

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
  maxFileSize: "100M"
  fileCount: "10"
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
  progress: <scheduled|running|finished|failure>
  node: <node name>
~~~




