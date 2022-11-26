# Network traffic capture in AKS Windows Nodes

# Objective

Getting Windows Pods Network Capture is usually a multi-step process. You need to connect to Node, install or run the necessary tools to accomplish this task, get the results through a Storage Account or kubectl cp. Using this script, all these steps are automated, you only need to download the results of your computer.

# Prerequisites

This script will deploy a manifest file with a privileged container on a Windows node and deploy the tcpdump.exe and azcopy.exe. We need followings:

kubectl binary connected to an active AKS cluster
SAS Key of a Blob Container with Write permission in order to upload the file. This will be exposed as an environment variable SAS, eg. export SAS="YourStorageAccountSAS-Key"

# Running the script

Copy/Paste the content of the script on your environment (bash), make it executable with

chmod +x ./kubectl-windumps

and run with the following command:

kubectl windumps akswin00001 60

where 60 represents the number of seconds of capture.

```
#!/bin/bash

if [ "$#" -lt 2 ];
then
  echo "Windows Nodes Network Capture"
  echo "Invalid number of arguments"
  echo "Usage: kubectl windumps nodeName captureTime(s)"
  echo "kubectl-windumps.sh akswin0001 30"

fi
nodeName="$1"
capTime="$2"

#Checking for existence of SAS Key as environment variable in SAS
if [[ -z "${SAS}" ]]; then
  echo "Storage Account Signature not found in environment variable. Please add the required value in SAS, eg. 'SAS=yourKey'"
  nodeName="$1"
  capTime="$2"
  SAS_key="$3"
  exit
else
cat << EOF > ./windumps.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: windows-debug-17263
  name: windows-debug-17263
  namespace: default
spec:
  nodeName: $1
  containers:
  - image: ghcr.io/jsturtevant/windows-debug:v1.0.0
    imagePullPolicy: Always
    name: windows-debug-17263
    resources: {}
    volumeMounts:
    - mountPath: /tmp
      name: logs
    command:
      - powershell 
      - Invoke-Webrequest https://www.microolap.com/downloads/tcpdump/tcpdump_trial_license.zip -useBasicParsing -OutFile C:\tmp\tcpdump.zip;
      - Invoke-WebRequest -UseBasicParsing https://aka.ms/downloadazcopy-v10-windows -OutFile c:\tmp\azcopy.zip;
      - Expand-Archive C:\tmp\tcpdump.zip -DestinationPath C:\tmp\; 
      - Expand-Archive c:\tmp\azcopy.zip -DestinationPath C:\tmp\;
      - c:\tmp\tcpdump -G $2 -W 1 -w c:\tmp\\$nodeName.pcap;
      - cd C:\tmp\azcopy*;
      - ./azcopy.exe copy "C:\tmp\\$nodeName.pcap" "$SAS";
      - sleep 3600
  volumes:
  - name: logs
    hostPath:
      path: /tmp
      type: Directory
  dnsPolicy: ClusterFirst
  hostNetwork: true
  nodeSelector:
    kubernetes.io/os: windows
  restartPolicy: Never
  securityContext:
    windowsOptions:
      hostProcess: true
      runAsUserName: NT AUTHORITY\SYSTEM
status: {}
EOF
echo "You can extract the capture file with the following command at the end of the tests: kubectl cp default/windows-debug-17263:/tmp/$nodeName.pcap ./$nodeName.pcap"

kubectl apply -f ./windumps.yaml

#kubectl cp default/windows-debug-17263:/tmp/akswinx000000.pcap ./akswinx000000.pcap


fi
```
