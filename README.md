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
