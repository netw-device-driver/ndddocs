# Installation

Network device driver is build using the k8s operator framework and extends k8s to configure network devices (like SRL) using k8s CRDs.

The next section outlines the steps to setup the network device driver framework. Although K8s is OS agnostic the steps outlined in the section are focussed on linux

## install a kind cluster

Install kind

```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.10.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

Create kind cluster with name ndd

```
kind create cluster --name ndd
```

Check if the cluster is running

```
kubectl get node
```

should give an output like this

```
NAME                STATUS   ROLES    AGE     VERSION
ndd-control-plane   Ready    master   2m58s   v1.19.1
```

The full documentation for installing a kind can be found here: [kind quickstart](https://kind.sigs.k8s.io/docs/user/quick-start/).

## install SRL lab

We typically opt to use [contaianerlab](https://containerlab.srlinux.dev) to spin up SR Linux containers. Of course other means can be used for setting up SRL containers; also real HW can be leveraged.

If you run containerlab on the same linux machine you should add the following rules in iptables to allow communication between the kind cluster network and container lab management network.
The bridges are specific to your environment

```
sudo iptables -I FORWARD 1 -i br-1283e519ba86 -o br-1f1c85a8b4c5 -j ACCEPT
sudo iptables -I FORWARD 1 -o br-1283e519ba86 -i br-1f1c85a8b4c5 -j ACCEPT
```

## install the network device controller

The network device controller is responsible for managining the device drivers and spins up ddriver deployment pods per network device.

Install network device controller and its respective CRD(s)

```
kubectl apply -f https://raw.githubusercontent.com/netw-device-driver/netw-device-controller/master/manifest.yaml
```

The above step creates a the following main elements in the k8s cluster:

Namespace: nddriver-system 

```
kubectl get ns
NAME                 STATUS   AGE
default              Active   20m
kube-node-lease      Active   20m
kube-public          Active   20m
kube-system          Active   20m
local-path-storage   Active   20m
nddriver-system      Active   14m
```

CRDs: 

```
kubectl get crd
NAME                             CREATED AT
devicedrivers.ndd.henderiw.be    2021-03-16T05:55:32Z
networkdevices.ndd.henderiw.be   2021-03-16T05:55:32Z
networknodes.ndd.henderiw.be     2021-03-16T05:55:32Z
```

- networknodes: defines the parameters to connect to the network device (which protocol to us, credentials, ip/port information)
- devicedrivers: allows to setup specific parameters that are relevant for the device driver container that you would use in your environment: e.g. a dedicated image can be used.

if the information supplied in the network nodes and device driver is successfull you will see tha

## Confgure the network node 

To setup the device driver we first need to configure a network node and optionally device driver parameters.

The following steps should be followed:

- deploy a secret: supplies username/password in base64 encoding
- deploy a network node:
    - name: this will be used in the ddriver deployment
    - device driver kind: gnmi, later we can use other protocols
    - target information:
        - address: ip address and port of the target network device
        - credential name: reference to the secret
        - encoding: json_ietf is used for now
        - skipVerify: skips certificate verification
- deploy a subscription profile for the device -> need to change this part

Example of configuration elements are supplied below

```
apiVersion: v1
kind: Secret
metadata:
  name: srl-secrets
type: Opaque
data:
  username: YWRtaW4K
  password: YWRtaW4K
```

```
apiVersion: ndd.henderiw.be/v1
kind: NetworkNode
metadata:
  name: leaf1
spec:
  device-driver:
    kind: gnmi
  target:
    address: 172.20.20.5:57400
    credentialsName: srl-secrets
    encoding: json_ietf
    skpVerify: true
```

```
apiVersion: ndd.henderiw.be/v1
kind: DeviceDriver
metadata:
  name: gnmi-device-driver
  labels:
    ddriver-kind: gnmi
spec:
  container:
    name: gnmi-dddriver
    image: henderiw/netwdevicedriver-gnmi:latest
    imagePullPolicy: Always
    command:
      - /netwdevicedriver-gnmi
    resources:
      requests:
          memory: "64Mi"
          cpu: "30m"
      limits:
        memory: "256Mi"
        cpu: "250m"
```

```
apiVersion: v1
data:
  subscriptions:
    /acl
    /bfd
    /interface
    /network-instance
    /platform
    /qos
    /routing-policy
    /tunnel
    /tunnel-interface
    /system/snmp
    /system/sflow
    /system/ntp
    /system/network-instance
    /system/name
    /system/mtu
    /system/maintenance
    /system/lldp
    /system/lacp
    /system/authentication
    /system/banner
    /system/bridge-table
    /system/ftp-server
    /system/ip-load-balancing
    /system/json-rpc-server
  excption-paths:
    interface[name=mgmt0]
    network-instance[name=mgmt]
    system/gnmi-server
    system/tls
    system/ssh-server
    system/aaa
    acl/cpm-filter
kind: ConfigMap
metadata:
  name: srl-k8s-subscription-config
  namespace: nddriver-system
```

If the configuration was successfull you should see a new deployment per network node/device.

```
kubectl get pods -n nddriver-system
NAME                                           READY   STATUS    RESTARTS   AGE
nddriver-controller-manager-5c9946596c-95zvz   2/2     Running   0          37m
nddriver-deployment-leaf1-579f48486f-5g7dl     1/1     Running   0          24s
```

The discovery parameters of the network device should be visible and in Ready state

```
kubectl describe networkdevices.ndd.henderiw.be
Name:         leaf1
Namespace:    default
Labels:       netwDevice=leaf1
Annotations:  <none>
API Version:  ndd.henderiw.be/v1
Kind:         NetworkDevice
Metadata:
  Creation Timestamp:  2021-03-16T06:33:04Z
  Generation:          1
  Managed Fields:
    API Version:  ndd.henderiw.be/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .:
          f:netwDevice:
      f:spec:
        .:
        f:address:
      f:status:
    Manager:      manager
    Operation:    Update
    Time:         2021-03-16T06:33:04Z
    API Version:  ndd.henderiw.be/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        f:discoveryStatus:
        f:hardwareDetails:
          .:
          f:kind:
          f:macAddress:
          f:serialNumber:
          f:swVersion:
        f:lastUpdated:
    Manager:         netwdevicedriver-gnmi
    Operation:       Update
    Time:            2021-03-16T06:33:10Z
  Resource Version:  10275
  Self Link:         /apis/ndd.henderiw.be/v1/namespaces/default/networkdevices/leaf1
  UID:               160f43b7-c5ac-4a6b-a273-34ecd1fcc12f
Spec:
  Address:  172.20.20.5:57400
Status:
  Discovery Status:  Ready
  Hardware Details:
    Kind:           7220 IXR-D2
    Mac Address:    00:01:00:FF:00:00
    Serial Number:  Sim Serial No.
    Sw Version:     v0.0.0
  Last Updated:     2021-03-16T06:33:10Z
Events:             <none>
```

## Install the SRL Provider/Operator

In order to use the system, a SRL provider/operator should be installed, which defines the CRD(s) to configure the SRL devices

```
kubectl apply -f https://raw.githubusercontent.com/netw-device-driver/srl-k8s-operator/master/manifest.yaml
```