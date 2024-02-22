# Load testing

This project is based on the [Performance-toolkit](https://github.com/magento/magento2/tree/2.4/setup/performance-toolkit) provided by Adobe.

However, this project is not maintained since 2021.
So we had to update the scenarios relevant for our project with Adobe commerce 2.4.6-p3 and update it for the French version.

## Prerequisite

* Go to the [Download Apache JMeter page](http://jmeter.apache.org/download_jmeter.cgi) and download JMeter in the Binaries section. Note that Java 8 or later is required.
  Unzip the archive.
* To use this we don't need to insert test data. It works with Chomette catalog and users.
* Configure the RMI server for SSL: [Documentation](https://jmeter.apache.org/usermanual/remote-test.html#setup_ssl)
* Launch the server(s): intances that will execute the scenarios on the destination server
  ```shell
  ./jmeter-server -Jserver.rmi.localport=[PORT] -Djava.rmi.server.hostname=[IPS]
  ```
* Launch the Master: instance that will send the load testing instructions to the servers
    ```shell
     ./jmeter -n -t [PATH_TO_JMX]/Benchmark.jmx -l [LOG_DEST_FOLDER]/Benchmark-result-$(date +"%Y-%m-%d-%H-%M-%S").jtl -e -o [HTML_REPORT_DEST_FOLDER]
   ```
  
  Example on local (all variables needs to be adjusted if you want to test yourself):  
  ```shell
  ./jmeter -n -t ./load-test/Benchmark.jmx \
  -Jhost=myproject.docker.localhost \
  -Jadmin_path=admin \
  -JfrontendPoolUsers=1 \
  -JadminPoolUsers=1 \
  -Jadmin_password=$ADMIN_PASSWORD \
  -Jadmin_user=$ADMIN_LOGIN \
  -Jroot_category_id=6 \
  -l ./load-test/results/tmc-$(date +"%Y-%m-%d-%H-%M-%S")/Benchmark-result.jtl \
  -e -o ./load-test/results/tmc-$(date +"%Y-%m-%d-%H-%M-%S")/
    ```

---

# Run from Rancher cluster

Prerequisite: 
```
### Kube ###

sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl

# Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the appropriate Kubernetes apt repository. If you want to use Kubernetes version different than v1.29, replace v1.29 with the desired minor version in the command below:
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Note: To upgrade kubectl to another minor release, you'll need to bump the version in /etc/apt/sources.list.d/kubernetes.list before running apt-get update and apt-get upgrade. This procedure is described in more detail in Changing The Kubernetes Package Repository.

# Update apt package index, then install kubectl:
sudo apt-get update
sudo apt-get install -y kubectl

### Helm ###

curl -fsSL https://baltocdn.com/helm/signing.asc | sudo gpg --dearmor -o /etc/apt/keyrings/helm.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm.list
sudo apt update
sudo apt install --assume-yes helm
```


### JMeter Helm chart: 
URL: https://artifacthub.io/packages/helm/cloudnativeapp/distributed-jmeter
### Helm Git project to checkout: 
URL: https://github.com/pedrocesarti/distributed-jmeter-docker
### Harbor (Image registry Jmeter 5.6): 
URL: https://registry.smile.fr/harbor/projects/310/summary
* Build image
```shell
docker build --build-arg JMETER_VERSION=5.6 . -t registry.smile.fr/mynamespace/jmeter-docker:5.6
```
* Push image
```shell
docker push registry.smile.fr/mynamespace/jmeter-docker:5.6
```
### Rancher Cluster: 
URL: https://rancher.smile.fr/dashboard/c/c-m-b7rw7sbp/explorer/pod
* Use smile rancher context:
```shell
kubectl config use-context mycontext
```
* change config of the cluster
```shell
helm upgrade jmeter [Helm Git project to checkout folder] --namespace tmc
```
* Deploy new benchark scenario on master node on cluser:
```shell
kubectl cp -n tmc [JMX_PATH]/Benchmark.jmx [MASTER_POD_NAME]:/jmeter/apache-jmeter-5.6/
```
* Connect to master pod
```shell
kubectl exec [MASTER_POD_NAME] -n tmc -it -- /bin/bash
```

## TESTS (rename the JMX file)
```shell
 export JAVA_OPTIONS="$JAVA_OPTIONS -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0" && ./jmeter -n -t ../Benchmark.jmx \
-Jhost=mywebsite.com \
-Jadmin_path=$ADMIN_PATH \
-Jramp_period=30 \
-JfrontendPoolUsers=50 \
-JadminPoolUsers=10 \
-JapiPoolUsers=30 \
-Jadmin_password=$ADMIN_PASS \
-Jadmin_user=$ADMIN_USER \
-Jroot_category_id=7 \
-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0
-l /tmp/tmc-$(date +"%Y-%m-%d-%H-%M-%S")/Benchmark-result.jtl \
-e -o /tmp/tmc-$(date +"%Y-%m-%d-%H-%M-%S")/
```
