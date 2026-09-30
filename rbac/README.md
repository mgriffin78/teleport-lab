## Setup GPU Dashboard reader ##
# Run the following commands to generate and sign certificate use to authenticate against the kubernetes API
```bash
#Create and sign certificate
openssl genrsa -out .certs/kubernetes/gpudash.key 2048
openssl req -new  -key .certs/kubernetes/gpudash.key  -out .certs/kubernetes/gpudash.csr -subj "/CN=mike/O=gpudash"
openssl x509 -req -in .certs/kubernetes/gpudash.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out .certs/kubernetes/gpudash.crt -days 500

#Add to kubernetes config and create context
kubectl config set-credentials mike --client-certificate=.certs/kubernetes/gpudash.crt  --client-key=.certs/kubernetes/gpudash.key
kubectl config set-context devops-context  --cluster=kubernetes --namespace=gpu-dashboard --user=mike

#Use context and validate current permissions under the dashboard-view role
kubectl config use-context devops-context
kubectl get pod
NAME                            READY   STATUS    RESTARTS       AGE
gpu-dashboard-788d958f8-jzmww   1/1     Running   3 (167m ago)   14h
gpu-dashboard-788d958f8-kk6tp   1/1     Running   3 (167m ago)   14h

kubectl auth whoami
ATTRIBUTE                                           VALUE
Username                                            mike
Groups                                              [gpudash system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [X509SHA256=2a4402f2a2114bbad51d15a615d23c5af371da6dbce5cfff0355f4b8c9dfd469]
