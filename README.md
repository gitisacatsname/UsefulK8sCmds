# Kubernetes and Tanzu useful commands

## Get last reason (event) why any pod is not state running
```bash
kubectl get pods -A | tail -n +2 | awk '{if ($4 != "Running") system ("echo ""; echo " $2 "; kubectl get events -o custom-columns=FirstSeen:.firstTimestamp,LastSeen:.lastTimestamp,Count:.count,From:.source.component,Type:.type,Reason:.reason,Message:.message --field-selector involvedObject.name=" $2 " -n " $1 " | tail -1")}'
```
![image](https://user-images.githubusercontent.com/94610393/211031957-d20f1fb6-425c-46f8-afd3-3097a3d47fb8.png)


## Agressive force deletion of all pods not currently "Running"
### You could change Running to Terminating to speedup uninstalling large apps with e.g. kapp, useful for labs
```bash
kubectl get pods -A | tail -n +2 | awk '{if ($4 != "Running") system ("kubectl -n " $1 " delete pods " $2 " --grace-period=0 " " --force ")}'
```

## Watch all pods in "realtime"
```bash
watch -n 0.5 kubectl get pods -A
```

## Force cluster recreation on Tanzu e.g. after updating ca in TKC object
### It adds a date based annotation which counts as changing the tkc object so cluster will gracefully rebuild
```bash
kubectl patch tkc tapmeifyoucan --type merge -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"
```

## Display "pulling images" left on cluster - big enough to sit on the couch. toilet, cowsay, fortune and figlet required. yep toilet is a cmd line tool.
#### On Debian based run once 
```zsh
sudo apt install toilet figlet cowsay fortune -y
```

```bash
while true;do;clear;toilet  -t -f bigmono12 --filter metal `kubectl get pods -A | tail -n +2 |  awk '{if ($4 != "Running") system ("echo ""; echo " $2 "; kubectl get events -o custom-columns=FirstSeen:.firstTimestamp,LastSeen:.lastTimestamp,Count:.count,From:.source.component,Type:.type,Reason:.reason,Message:.message --field-selector involvedObject.name=" $2 " -n " $1 " | tail -1")}' | grep Pulling | wc -l` imgs pulling;fortune | cowsay | toilet  -t -f smbraille ;sleep 30;done
```

checks if kubectl apply second command (i know, haha, bad)
```bash
while true;do;clear;toilet  -t  --filter metal `k apply -f clusterissuer.yml > /dev/null; k apply -f cert.yaml > /dev/null;if [ $? -eq 0 ];then;echo "issuer status exited successfully";else;echo "issuer status exited with error code";fi`; fortune | cowsay | toilet  -t -f smbraille ;sleep 30;done
```

![image](https://user-images.githubusercontent.com/94610393/211031776-8768bcac-9cac-4d2d-94a8-874ebddb2272.png)


## Create node with bigger container image storage in tanzu by editing TKC object, useful if exp. disk pressure on low count vertical node setup with big container images

```yaml
workers:
  count: 3
  class: best-effort-medium
  storageClass: vwt-storage-policy
  volumes:
    - name: containerd
      mountPath: /var/lib/containerd
      capacity:
        storage: 16Gi 
```

## pause reconciliation of kapp, reinstall component to force update

learningcenter is not rebuild when running update on tap full profile - can be used to reinstall learningcenter after install of full profile in tap

first pause
```bash
kubectl patch pkgi tap -n tap-install -p '{"spec":{"paused":true}}' --type=merge
```

then uninstall 
```bash 
tanzu package installed delete learningcenter-workshops -n tap-install
```

run install command or update to reload values file 
```bash 
tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.3.4 --values-file tap.yml -n tap-install
```


## un/pause reconciliation of kapp

```bash
kubectl patch pkgi tap -n tap-install -p '{"spec":{"paused":false}}' --type=merge
kubectl patch pkgi tap-gui -n tap-install -p '{"spec":{"paused":false}}' --type=merge
```

```bash
kubectl patch pkgi learningcenter -n tap-install -p '{"spec":{"paused":false}}' --type=merge
kubectl patch pkgi learningcenter-workshops -n tap-install -p '{"spec":{"false":true}}' --type=merge
```

```bash
kubectl patch pkgi learningcenter -n tap-install -p '{"spec":{"paused":true}}' --type=merge
kubectl patch pkgi learningcenter-workshops -n tap-install -p '{"spec":{"paused":true}}' --type=merge
```

```bash
kubectl patch pkgi tap -n tap-install -p '{"spec":{"paused":true}}' --type=merge
kubectl patch pkgi tap-gui -n tap-install -p '{"spec":{"paused":true}}' --type=merge
```

## tap docker patch to build docs in gui - pause reconciliation first as shown above - or it will reset by kapp - use with care

```bash
kubectl patch deploy server -n tap-gui --patch-file tap-gui-dind-patch.yaml
```

delete gui pods
```bash
kubectl delete pod -l app=backstage -n tap-gui
```

tap-gui-dind-patch.yaml:
```yaml
spec:
  template:
    spec:
      containers:
      - command:
        - dockerd
        - --host
        - tcp://127.0.0.1:2375
        image: docker:dind-rootless
        imagePullPolicy: IfNotPresent
        name: dind-daemon
        resources: {}
        securityContext:
          privileged: true
          runAsUser: 0
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /tmp
          name: tmp
        - mountPath: /output
          name: output
      - name: backstage
        env:
        - name: DOCKER_HOST
          value: tcp://localhost:2375
        volumeMounts:
        - mountPath: /tmp
          name: tmp
        - mountPath: /output
          name: output
      volumes:
      - emptyDir: {}
        name: tmp
      - emptyDir: {}
        name: output%                                         
```


### patch workshop image with cas from node

stop recon. learningcenter workshop patch:

```yaml
spec:
  session:
    env:
    - name: NODE_EXTRA_CA_CERTS
      value: "/etc/pki/tls/certs/ca-bundle.crt"
```

patch dockerincontainer image

## tap full profile with ext certs

```yaml
shared:
  ingress_domain: "k8s.fullauto.cloud"
  image_registry:
    project_path: "harbor.k8s.fullauto.cloud/tap/supply-chain"
    username: "admin"
    password: "Password!"
  kubernetes_distribution: "" # To be passed only for OpenShift. Defaults to "".
  ca_cert_data: |
      -----BEGIN CERTIFICATE-----
      MIIDKDCCAhCgAwIBAgIQZbZZRefGKAKTJmuWc86FmjANBgkqhkiG9w0BAQsFADAU
      DwmZTI9cZ942hT+pqsGrak0zXrbfOeTgnju00000QR93ERQMGTt6GdelTuw=
      -----END CERTIFICATE-----

ceip_policy_disclosed: true # Installation fails if this is not set to true. Not a string.

#The above keys are minimum numbers of entries needed in tap-values.yaml to get a functioning TAP Full profile installation.

#Below are the keys which may have default values set, but can be overridden.

profile: full # Can take iterate, build, run, view.

excluded_packages:
- policy.apps.tanzu.vmware.com

supply_chain: testing_scanning # Can take testing, testing_scanning.

contour:
  envoy:
    service:
      type: LoadBalancer # This is set by default, but can be overridden by setting a different value.

buildservice:
  kp_default_repository: "harbor.k8s.fullauto.cloud/tap/build-service"
  kp_default_repository_username: "admin"
  kp_default_repository_password: "Password!"

tap_gui:
  app_config:
    catalog:
      locations:
        - type: url
          target: https://github.com/gitisapetssname/catalog/blob/main/catalog-info.yaml
  service_type: ClusterIP # If the shared.ingress_domain is set as above, this must be set to ClusterIP.
  tls:
    namespace: tap-gui
    secretName: tap-gui-cert
    
learningcenter:
  ingressSecret:
    certificate: |
      -----BEGIN CERTIFICATE-----
      MIIEEjCCAvqgAwIBAgIUfNVt9/PHPy408ZALujiFBMjJQu0wDQYJKoZIhvcNAQEL
      BQAwgY8xCzAJBgNVBAYTAkRFMQ8wDQYDVQQIDAZIZXNzZW4xDzANBgNVBAcMBk11
      r0Kmbitm56tW61Dod9GV9zayJOCVK8WCuGAu4r+szqZNq31Ophg=
      -----END CERTIFICATE-----
    privateKey:  |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQE00000N2w6K6MApTh0QdYra4aDxH2abO9u1Z9UEPYv4fPb
      GogeBLt97ragEisO5hu0ZII40pkqvDAW5rdU0000000oJ19pKUZk
      -----END RSA PRIVATE KEY-----

metadata_store:
  ns_for_export_app_cert: "dev"
  app_service_type: ClusterIP # Defaults to `LoadBalancer`. If `shared.ingress_domain` is set as above, this must be set to `ClusterIP`.

scanning:
  metadataStore:
    url: "" # Configuration is moved, so set this string to empty.

grype:
  namespace: "dev"
  targetImagePullSecret: "regcred"
```

## list all fonts in toilet

```bash
for f in /usr/share/figlet/* 
do 
  fs=$(basename $f)
  fname=${fs%%.tlf}
  toilet -f $fname $fname
done
```

![image](https://user-images.githubusercontent.com/94610393/211032204-0e8aecc0-1dd8-421a-9d6d-4987cfa27626.png)
