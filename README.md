# Kubernetes and Tanzu useful commands

## Get last reason (event) why any pod is not state running
kubectl get pods -A | awk '{if ($4 != "Running") system ("echo ""; echo " $2 "; kubectl get events -o custom-columns=FirstSeen:.firstTimestamp,LastSeen:.lastTimestamp,Count:.count,From:.source.component,Type:.type,Reason:.reason,Message:.message --field-selector involvedObject.name=" $2 " -n " $1 " | tail -1")}'

## Agressive force deletion of all Pods not currently "Running"
kubectl get pods -A | awk '{if ($4 != "Running") system ("kubectl -n " $1 " delete pods " $2 " --grace-period=0 " " --force ")}'

## Watch all Pods in "realtime"
watch -n 0.5 kubectl get pods -A

## Force Cluster recreation Tanzu e.g. after updating ca in TKC object
### It adds a date based annotation which counts as changing the tkc object so cluster will gracefully reinit
kubectl patch tkc tapmeifyoucan --type merge -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"

## Display Pulling Images left on Cluster big enough to sit on the couch. toilet required. yep toilet is a cmd line tool.
while true
do
toilet -f bigascii12 `kubectl get pods -A |  awk '{if ($4 != "Running") system ("echo ""; echo " $2 "; kubectl get events -o custom-columns=FirstSeen:.firstTimestamp,LastSeen:.lastTimestamp,Count:.count,From:.source.component,Type:.type,Reason:.reason,Message:.message --field-selector involvedObject.name=" $2 " -n " $1 " | tail -1")}' | grep Pulling | wc -l` images currently pulling;sleep 10
done

