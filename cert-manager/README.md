## Install Cert Manager ##
```bash
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.2 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```
# Create Certificate using openssl #
```bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout private.key -out certificate.crt -days 365
..+..+......+.+........+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+.........+......+.....+.............+..+.+..+......+.+...+...+........+....+...+..+.+.........+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...........+...+.............+......+........+......+.+.....+.+.....+...+.+...+..+...................+.....+................+.........+.........+...+........+......+...+.+...........+...+......+.+...........+....+...+...+..............+.+...+...............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
..............+...+..+....+.....+....+............+......+..+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+...............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*........+.+..+.........+.+..+...+...+.+...........+..........+..+..........+..+..........+......+........+....+.................+.+...+......+...+...+..................+...........+...+............+...+..........+......+..+..................+.+......+..+......+....+..+......+....+...+..+....+.....+..........+..+...................+........+.+...........+...+.+...........+....+..+...+......+...+..........+......+...........+...+...+............+...+..........+..+............+.+...............+...+............+..+.......+......+...+............+.....+.......+..+..........+.........+.....+....+..+.+..+.+.....+......+..................+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:California
Locality Name (eg, city) []:Livermore
Organization Name (eg, company) [Internet Widgits Pty Ltd]:teportation
Organizational Unit Name (eg, section) []:dev
Common Name (e.g. server FQDN or YOUR name) []:gpu-dashboard.com
Email Address []:
```

# Create Secret #
```bash
kubectl create secret tls dashboard-secret \
  --cert=certificate.crt \
  --key=private.key \
  --namespace=gpu-dashboard
```

# Apply Issuer and Certificate
```bash
root@ubuntu-01:~/rbac# kubectl apply -f gpu-certificate .yaml
root@ubuntu-01:~/rbac# kubectl apply -f gpu-certificate.yaml
```

