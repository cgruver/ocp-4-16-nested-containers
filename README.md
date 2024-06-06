# ocp-4-16-nested-containers
Test Repo for OCP 4.16 User Namespaces

## OpenShift Config Changes for UserNamespace and ProcMountType Support

```bash
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: ContainerRuntimeConfig
metadata:
 name: enable-crun-master
spec:
 machineConfigPoolSelector:
   matchLabels:
     pools.operator.machineconfiguration.openshift.io/master: ""
 containerRuntimeConfig:
   defaultRuntime: crun
EOF

oc patch FeatureGate cluster --type merge --patch '{"spec":{"featureSet":"CustomNoUpgrade","customNoUpgrade":{"enabled":["ProcMountType","UserNamespacesSupport"]}}}'

cat << EOF | butane | oc apply -f -
variant: openshift
version: 4.15.0
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: nested-podman-master
storage:
  files:
  - path: /etc/crio/crio.conf.d/99-nested-podman
    mode: 0644
    overwrite: true
    contents:
      inline: |
        [crio.runtime]
        allowed_devices = [
          "/dev/fuse",
          "/dev/net/tun"
        ]
  - path: /etc/modules-load.d/nested-podman.conf
    mode: 0644
    overwrite: true
    contents:
      inline: |
        tun
        rtnl-link-bridge
        ipt_addrtype
EOF
```

```bash
cat << EOF | oc apply -f -
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: nested-podman-scc
priority: null
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities:
- SETUID
- SETGID
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
groups: []
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
runAsUser:
  type: MustRunAsRange
  uidRangeMax: 20000
  uidRangeMin: 10000
seLinuxContext:
  type: MustRunAs
  seLinuxOptions:
    type: container_engine_t
supplementalGroups:
  type: RunAsAny
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
- hostPath
EOF
```

```bash
module nested_containers 1.0;

require {
	type null_device_t;
	type zero_device_t;
	type urandom_device_t;
	type random_device_t;
	type container_file_t;
	type container_engine_t;
	type devtty_t;
	type setfiles_t;
	class chr_file { mounton read setattr write };
	class sock_file mounton;
}

#============= container_engine_t ==============
allow container_engine_t container_file_t:sock_file mounton;
allow container_engine_t devtty_t:chr_file mounton;
allow container_engine_t null_device_t:chr_file { mounton setattr };
allow container_engine_t random_device_t:chr_file mounton;
allow container_engine_t urandom_device_t:chr_file mounton;
allow container_engine_t zero_device_t:chr_file mounton;
```

```bash
cat << EOF | oc apply -f -
apiVersion: v1                      
kind: Namespace                 
metadata:
  name: devspaces
---           
apiVersion: org.eclipse.che/v2 
kind: CheCluster   
metadata:              
  name: devspaces  
  namespace: devspaces
spec:                         
  components:                  
    cheServer:      
      debug: false
      logLevel: INFO
    metrics:                
      enable: true
    pluginRegistry:
      openVSXURL: https://open-vsx.org
  containerRegistry: {}      
  devEnvironments:       
    startTimeoutSeconds: 300
    secondsOfRunBeforeIdling: -1
    maxNumberOfWorkspacesPerUser: -1
    maxNumberOfRunningWorkspacesPerUser: 5
    containerBuildConfiguration:
      openShiftSecurityContextConstraint: nested-podman-scc
    disableContainerBuildCapabilities: false
    defaultComponents:
    - name: dev-tools
      container:
        image: quay.io/cgruver0/che/dev-tools:latest
        memoryLimit: 6Gi
        mountSources: true
    defaultEditor: che-incubator/che-code/latest
    defaultNamespace:
      autoProvision: true
      template: <username>-devspaces
    secondsOfInactivityBeforeIdling: 1800
    storage:
      pvcStrategy: per-workspace
  gitServices: {}
  networking: {}   
EOF
```

## Run the Demos

1. Create an Eclipse Che workspace from this git repo: `https://github.com/cgruver/openshift-nested-containers.git`

### Run a container:

1. Open a terminal into the `openshift-nested-containers` project.

1. Clear the `CONTAINER_HOST` environment variable: (We're going to run this without a "remote" podman socket)

   ```bash
   unset CONTAINER_HOST
   ```

1. Run the following:

   ```bash
   podman run -d --rm --name webserver -p 8080:80 quay.io/libpod/banner
   curl http://localhost:8080
   ```

1. Run an interactive container:

   ```bash
   podman run -it --rm registry.access.redhat.com/ubi9/ubi-minimal
   ```

### Demo of Ansible Navigator

1. Clear the `CONTAINER_HOST` environment variable: (We're going to run this without a "remote" podman socket)

   ```bash
   unset CONTAINER_HOST
   ```

1. Open a terminal into the `ansible-demo` project.

1. Run the following command:

   ```bash
   ansible-navigator run helloworld.yaml
   ```

### Demo of AWS SAM CLI with Podman

1. Use the `Task Manager` extension to run the `Start Podman Service` task.

1. Open a terminal in the `sam-cli-demo` project:

   ```bash
   sam init --name sam-py-app --architecture=x86_64 --location="./python3.9/hello" --no-tracing --no-application-insights --no-input
   cd sam-py-app
   sam build
   sam local start-api --debug --docker-network=podman
   ```

1. Open a second terminal to invoke the Lambda function:

   ```bash
   curl http://127.0.0.1:3000/hello
   ```

### Demo of Quarkus Dev Services with Podman

1. Open a terminal in the `quarkus-dev-services-demo` project:

   ```bash
   mvn clean
   mvn test
   ```

## Demo of AWS Dev With Localstack with Podman

1. Use the `Task Manager` extension to run the `Start Localstack Service` task.

1. Open a terminal in the `localstack-demo` project:

   ```bash
   make deploy
   awslocal s3 ls s3://archive-bucket/
   ```
