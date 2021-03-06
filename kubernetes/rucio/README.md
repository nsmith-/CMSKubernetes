Adapted from rucio instructions here: https://github.com/rucio/helm-charts/tree/master/rucio-server


# First time setup

## OpenStack project

You need to request a personal OpenStack project in which to install your kubernetes cluster. 
You might also want to request a quota for "Shares" in the "Geneva CephFS Testing" type for persistent data storage. 
This is used right now by Graphite.

## Setup a new cluster in the CMSRucio project:

Begin by logging into the CERN cloud infrastructure `slogin lxplus7-cloud.cern.ch` then:

    openstack coe cluster delete --os-project-name CMSRucio  cmsruciotest
    openstack coe cluster create cmsruciotest --keypair lxplus  --os-project-name CMSRucio   --cluster-template kubernetes-preview --node-count 4
    openstack coe cluster list --os-project-name CMSRucio # Monitor creation status

If you are creating your own project for development, please omit `--os-project-name CMSRucio`. 
This will create a kubernetes cluster in your own openstack space rather than the central group space.
CMSRucio is a project space CERN has setup for us to contain our production and testbed servers.

### If setting up a new/changed cluster:

    cd [some directory] # or use $HOME
    export BASEDIR=`pwd`
    rm key.pem cert.pem ca.pem config
    openstack coe cluster config  --os-project-name CMSRucio  cmsruciotest
    
This command will show you the proper value of your `KUBECONFIG` variable which should be this:   
    
    export KUBECONFIG=$BASEDIR/config


Copy and paste the last line. On subsequent logins it is all that is needed. Now make sure that your nodes exist:

    -bash-4.2$ export KUBECONFIG=[as above]
    -bash-4.2$ kubectl get nodes
    NAME                                 STATUS    ROLES     AGE       VERSION
    cmsruciotest-mzvha4weztri-minion-0   Ready     <none>    5m        v1.11.2
    cmsruciotest-mzvha4weztri-minion-1   Ready     <none>    6m        v1.11.2
    cmsruciotest-mzvha4weztri-minion-2   Ready     <none>    6m        v1.11.2
    cmsruciotest-mzvha4weztri-minion-3   Ready     <none>    6m        v1.11.2

## Setup the helm repos if needed

    export KUBECONFIG=[as above]
    kubectl config current-context
    helm repo add rucio https://rucio.github.io/helm-charts
    helm repo add kiwigrid https://kiwigrid.github.io

## Label nodes for ingress and add the same nodes to the DNS registration

Pick at least two of the nodes from the output of the `kubectl get nodes` command above. (Two for load balancing/redundancy.)

    kubectl label node cmsruciotest-mzvha4weztri-minion-0 role=ingress
    kubectl label node cmsruciotest-mzvha4weztri-minion-3 role=ingress
    openstack server set  --os-project-name CMSRucio  --property landb-alias=cms-rucio-test--load-1- cmsruciotest-73m6rlb5qg4p-minion-0 
    openstack server set  --os-project-name CMSRucio  --property landb-alias=cms-rucio-test--load-2- cmsruciotest-73m6rlb5qg4p-minion-3

`openstack server unset` will undo this. One must wait up to 15 minutes after the openstack commands for the DNS registration to become active. More details are here: https://clouddocs.web.cern.ch/clouddocs/containers/tutorials/lb.html

Repeat this process for the graphite ingress node

    kubectl label node cmsruciotest-73m6rlb5qg4p-minion-1 role=ingress
    openstack server set  --os-project-name CMSRucio  --property landb-alias=cms-rucio-graphite-test--load-1- cmsruciotest-73m6rlb5qg4p-minion-1

## Setup StorageClass (if needed)

If we have CephFS storage needs, one makes a single storage class for the entire claim:

    kubectl create -f cms-rucio-storage.yaml

## Install helm into the kubernetes project

    helm init
    kubectl create serviceaccount --namespace kube-system tiller
    kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
    kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

## Get a host certificate for the server

Go to ca.cern.ch and get the host certificate for one of the load balanced names above. 
Add Service alternate names for the load balanced hostname, e.g. cms-rucio-dev.cern.ch and cms-rucio-dev

Download this certificate file and use the relevant openssl commands to create tls.crt and tls.key (exactly those names)

    openssl pkcs12 -in cms-rucio-dev--load-1-.p12 -clcerts -nokeys -out ./tls.crt
    openssl pkcs12 -in cms-rucio-dev--load-1-.p12 -nocerts -nodes -out ./tls.key

## Create relevant secrets 

Create the other secrets with (on lxplus7-cloud.cern.ch and/or other machines):

    kubectl create secret tls rucio-server.tls-secret --key=tls.key --cert=tls.crt # Same as just created above
    kubectl create secret generic fts-cert --from-file=[robot certificate] # Must be named usercert.pem
    kubectl create secret generic fts-key --from-file=[robot key, unencrypted] # Must be named new_userkey.pem
    kubectl create secret generic hermes-cert --from-file=[robot certificate] # Same as for FTS
    kubectl create secret generic hermes-key --from-file=[robot key, unencrypted] # Same as for FTS
    kubectl create secret generic cms-ruciod-testbed-rucio-ca-bundle --from-file=/etc/pki/tls/certs/CERN-bundle.pem
    kubectl create secret generic cms-analysisd-testbed-rucio-ca-bundle --from-file=/etc/pki/tls/certs/CERN-bundle.pem
    ./CMSKubernetes/kubernetes/rucio/rucio_reaper_secret.sh  # Currently needs to be done after helm install below 
    kubectl get secrets

N.b. In the past we also used /etc/pki/tls/certs/CERN-bundle.pem as a volume mount for logstash. 
That no longer seems to be needed.

## Install CMS server into the kubernetes project. Later we can add another set of values files for testbed, integration, production

    export KUBECONFIG=[as above]
    cd CMSKubernetes/kubernetes/rucio
    ./install_rucio_[production, testbed, etc].sh

After helm has installed everything and the fts-cert and fts-key secretes are made, 
the daemons still won't have started because the CronJob which creates the proxy secret will not have run.
Fix this by manually running the proxy generating job:

    kubectl create job --from=cronjob/cms-ruciod-testbed-renew-fts-proxy fts1
    kubectl create job --from=cronjob/cms-analysisd-testbed-renew-fts-proxy fts2

# To upgrade the servers

The above is what is needed to get things bootstrapped the first time. After this, you can modify the various yaml files and

    export KUBECONFIG=[as above]
    cd CMSKubernetes/kubernetes/rucio
    ./upgrade_rucio_[production, testbed, etc].sh
    
## Upgrade helm if needed

CERN has added helm to the lxplus-cloud machines. 
If you get a version incompatibility warning from helm, 
you may need to upgrade tiller (installed in kubernetes) with 
    
    helm init --upgrade
    
# Use a node outside of kubernetes as an authorization server and to delegate FTS proxies

While we expect that eventually the authorization server will be able to run inside of kubernetes, at the moment it cannot.
This is because of how kubernetes/traefik handles (or doesn't) client certificates. 
For the moment, it's easiest just to turn an OpenStack node into the authorization server. 
The same VM can be used to manage the FTS proxies.

## Setup the VM node itself

This recipe is for starting with an OpenStack CC7 node which we assume is named `cms-rucio-authz`. 
The OpenStack "small" type is fine.

### Get a host certificate installed

Use CERN CA https://ca.cern.ch/ca/ to generate a host cert, 
scp the cert to lxplus:~/.globus/ and then create certificate and key:

    sudo mkdir /etc/grid-security
    sudo openssl pkcs12 -in ~/.globus/cms-rucio-authz.p12  -clcerts -nokeys -out /etc/grid-security/hostcert.pem
    sudo openssl pkcs12 -in ~/.globus/cms-rucio-authz.p12   -nocerts -nodes -out /etc/grid-security/hostkey.pem
    sudo chmod 0600 /etc/grid-security/hostkey.pem

Become root (`sudo su`) and install some general packages and docker:

    yum install -y yum-utils device-mapper-persistent-data lvm2 nano git
    yum update -y
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    yum install -y docker-ce
    systemctl start docker
    systemctl enable docker
    docker run hello-world
    usermod -a -G docker [your user name]

Continuing as root, add what's needed to accept proxies for the auth server:

    yum -y install voms-clients-cpp
    yum -y install http://linuxsoft.cern.ch/wlcg/centos7/x86_64/wlcg-repo-1.0.0-1.el7.noarch.rpm
    curl -o  /etc/yum.repos.d/ca.repo https://raw.githubusercontent.com/rucio/rucio/master/etc/docker/dev/ca.repo
    yum update 
    yum -y install ca-certificates.noarch lcg-CA voms-clients-cpp wlcg-voms-cms fetch-crl 
    systemctl start fetch-crl-cron
    systemctl enable fetch-crl-cron 
    exit
    
## Start (or restart) the authorization server docker image

    ./CMSKubernetes/kubernetes/rucio/start_rucio_auth.sh  # Or similar depending on which auth server you want to start
   
# Get a client running and connect to your server

It can be easiest just to create another container with a client installed to connect to your new server. 
There is a client YAML file that can also be installed into your newly formed kubernetes cluster. 
Find the client name below from the output of `kubectl get pods`

    kubectl create -f rucio-client.yaml 
    kubectl exec -it client-6c4466d746-gwl9g /bin/bash
    
And then in your client container, setup a proxy or use user/password etc. and
    
    export RUCIO_ACCOUNT=[an account your identity maps to]
    rucio whoami 

From here you should be able to use any rucio command line commands you need to.

# Connect to graphite from a web browser

This is complicated at the moment. One needs to figure out what node the server is exposed on using kubectl tools and 
use tunneling if connecting from outside of CERN. 

Setup the tunnel with 

    ssh -D 8089 -N ${USER}@lxplus.cern.ch

and configure your browser appropriately (Firefox works well with SOCK5 proxies) then visit the URL for the graphite server
which at the time of writing was http://cmsruciotest-73m6rlb5qg4p-minion-3.cern.ch:30862 
