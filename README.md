# awx-on-k3s
Ansible AWX setup on k3s


Configure mariadb and create database k3s:

    grant all privileges on `k3s`.* to `k3`@`%`;
    grant usage on k3s.* to 'k3s'@'%' identified by 'secret';



# Install k3s - servers

Define the datastore:

    export K3S_DATASTORE_ENDPOINT='mysql://k3s:secret@tcp(10.0.0.2:3306)/k3s'


Download and install:

    curl -sfL https://get.k3s.io | sh -s - server --node-taint CriticalAddonsOnly=true:NoExecute --tls-san 10.0.0.3 --tls-san X.X.X.X


Find the token:

    cat /var/lib/rancher/k3s/server/node-token
    
Add the second node:

    curl -sfL https://get.k3s.io | sh -s - server --node-taint CriticalAddonsOnly=true:NoExecute --tls-san 10.0.0.3 --tls-san X.X.X.X --token <token>
   
    

## Install k3s - agents

On every VM run this:

    curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.3:6443 K3S_TOKEN=<token> sh -
    
    
## Testing

Check the nodes:

```
root@s-1:# kubectl get nodes
NAME      STATUS   ROLES                  AGE     VERSION
s-1       Ready    control-plane,master   19m     v1.21.5+k3s1
s-2       Ready    control-plane,master   11m     v1.21.5+k3s1
agent-1   Ready    <none>                 2m15s   v1.21.5+k3s1
agent-2   Ready    <none>                 68s     v1.21.5+k3s1
agent-3   Ready    <none>                 35s     v1.21.5+k3s1
```


## Configure kubectl 

copy file `/etc/rancher/k3s/k3s.yaml` from one of the server into `~/.kube/config`

Edit the config file and change the `server` with the IP of the load balancer:

```
apiVersion: v1
clusters:
- cluster:
    ...
    server: https://X.X.X.X:6443
    ...
```

## Install k8s dashboard

Install: (https://github.com/kubernetes/dashboard/releases)

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml

Create admin an role:

The admin file: `dashboard.admin-user.yml`:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

The role file: `dashboard.admin-user-role.yml`:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Then run:

     kubectl create -f dashboard.admin-user.yml -f dashboard.admin-user-role.yml
     
     
 Get the token:
 
     kubectl -n kubernetes-dashboard describe secret admin-user-token | grep '^token'


Create a proxy:

    $ kubectl proxy
    Starting to serve on 127.0.0.1:8001
    
Login using the token here: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login

