# Kubernetes and Tanzu useful commands

## Get last reason (event) why any pod is not state running
kubectl get pods -A | awk '{if ($4 != "Running") system ("echo ""; echo " $2 "; kubectl get events -o custom-columns=FirstSeen:.firstTimestamp,LastSeen:.lastTimestamp,Count:.count,From:.source.component,Type:.type,Reason:.reason,Message:.message --field-selector involvedObject.name=" $2 " -n " $1 " | tail -1")}'

## Delete all error Pods now and force it
kubectl get pods -A | awk '{if ($4 != "Running") system ("kubectl -n " $1 " delete pods " $2 " --grace-period=0 " " --force ")}'

## Watch all Pods in "realtime"
watch -n 0.5 kubectl get pods -A

## Force Cluster recreation Tanzu e.g. after updating ca in TKC object
### It adds a date based annotation which counts as changing the tkc object so cluster will gracefully reinit
kubectl patch tkc tapmeifyoucan --type merge -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"
