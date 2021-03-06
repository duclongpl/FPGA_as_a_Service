# k8s-fpga-device-plugin
## About
The Xilinx FPGA device plugin for Kubernetes is a Daemonset deployed on the kubernetes(a.k.a k8s) cluster which allows you to:

* Discover the FPGAs inserted in each node of the cluster and expose info of the FPGAs such as quantities, DSA(shell) type and timestamp, etc
* Run FPGA accessible containers in the k8s cluster

More info about k8s device plugin, please refer to https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/

## Prerequisites
* All FPGAs have the DSA(shell) flashed already.
* Xilinx FPGA device drivers installed through XRT on each node of the cluster where there are FPGA(s) inserted (drivers installed through xbinst would not work)
* Container runtime in k8s is docker
* k8s version = 1.12 (all tests have been running with this version. Old version may or may not work)

## Quick start
Assume you have a running k8s cluster already.

All cmds mentioned in this part run on the master node of k8s cluster

### Enable Xilinx FPGA support in k8s
#### Deploy FPGA device plugin as daemonset
```
$kubectl create -f fpga-device-plugin.yml
```
#### Check nodes status and the FPGA resource status on the node
```
$kubectl get node
$kubectl describe node <node_name>
```
### Run jobs accessing FPGA
The Xilinx FPGA resources all have name with following format

xilinx.com/fpga-dsatype-timestamp

eg. xilinx.com/fpga-xilinx_kcu1500_dynamic_5_0-1512563160

The exact name of the FPGA resource on each node can be extracted from the output of
```
$kubectl describe node <node_name>
```
A user pod requesting FPGA resource can be deployed now.

#### Deploy user pod
```
$kubectl create -f mypod.yaml
```
```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-pod
    image: xilinxatg/fpga-verify:2018.10.29
  resources:
    limits:
      xilinx.com/fpga-xilinx_kcu1500_dynamic_5_0-1512563160: 2
  volumeMounts:
    - name: hostopt
      mountPath: /opt
      readOnly: true
  command: ["/bin/sh"]
  args: ["-c", "while true; do echo hello; sleep 5;done;"] 
volumes:
  - name: hostopt
  hostPath:
    path: /opt
```
#### Check status of the deployed pod
```
$kubectl get pod
$kubectl describe pod <pod_name>
```
#### Run hello world in the pod
```
$kubectl exec -it mypod /bin/bash
mypod>source /opt/xilxinx/xrt/setup.sh
mypod>xbutil scan
mypod>/tmp/verify.exe /tmp/1500/verify.xclbin -d 0
mypod>/tmp/verify.exe /tmp/1500/verify.xclbin -d 1
```
In this test case, the container image (xilinxatg/fgpa-verify:2018.10.29) has been pushed to docker hub. It can be publicly accessed

The image contains verify.xclbin for many types of FPGA, please select the type matching the FPGA resource the pod requests. 

## Known issues
* When there are multiple types of FPGA on one node, the device plugin registers resource for each
  specific type. The k8s device plugin framework has issue handling this case. I have filed issue report tracking this. https://github.com/kubernetes/kubernetes/issues/70350
* When there are multiple FPGAs of same type on one node, if a pod requests just partial FPGAs(not
  all), XRT 2018.2 release has issue – 'xbutil scan' still lists all FPGAs within container although
  not all are assigned to the container, while the 'xbutil list' can only access the assigned
  FPGA(s), so the index of the FPGAs gets messed and the app is confused to know which FPGA to use.
  This has been fixed in XRT 2018.3 release

