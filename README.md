# Kubernetes and Tanzu useful commands

## Get last reason (event) why any pod is not state running
```bash
kubectl get pods -A | tail -n +2 | awk '{if ($4 != "Running") system ("echo ""; echo " $2 "; kubectl get events -o custom-columns=FirstSeen:.firstTimestamp,LastSeen:.lastTimestamp,Count:.count,From:.source.component,Type:.type,Reason:.reason,Message:.message --field-selector involvedObject.name=" $2 " -n " $1 " | tail -1")}'
```
![image](https://user-images.githubusercontent.com/94610393/211031957-d20f1fb6-425c-46f8-afd3-3097a3d47fb8.png)


## Agressive force deletion of all pods not currently "Running"
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
