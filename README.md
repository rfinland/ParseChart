![Parse Evolution](images/logo.jpg)

# Parse Evolution
Check docker-compose config:
```bash
git clone https://github.com/rfinland/ParseChart.git
cd ParseChart
docker-compose config 
```
Notice: Application-Id values are selective with user MASTER_KEY; you can set values for these envs as you want.
In our case APP_ID=MyParseApp and MASTER_KEY=adminadmin in parse 
Up the app:
```bash
docker-compose up -d
#OR If already exist:
docker-compose up --force-recreate -d
```
#notice: If you want to create kube via docker-compose yaml file you have to set your envs as static in your yaml file
#Check Parse status
```bash
curl http://localhost:1337/parse/health
#Result: {"status":"ok"}
```
# POST 
Post a record:
You have to set your X-Parse-Application-Id as you set in environment section on your docker-compose file (in our case:APP_ID=MyParseApp)
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://localhost:1337/parse/classes/GameScore
```
You should get a response similar to this:
```bash
{
"objectId":"ngaYmcDv1m",
"createdAt":"2021-12-06T06:56:09.068Z"
}
```
# GET
Get the record:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:1337/parse/classes/GameScore/ngaYmcDv1m
```
Get all records that you defined:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:1337/parse/classes/GameScore
```
Also you can see the result in your browser (you just need set X-Parse-Application-Id: MyParseApp )
# Recommended add-ons
##### JSON Formatter: 
Chrome extension for printing JSON and JSONP nicely when you visit it 'directly' in a browser tab.
[JSON Formatter](https://github.com/callumlocke/json-formatter)

##### Modify Header:
Add, modify or remove a header for any request on desired domains.
[Modify Header](https://mybrowseraddon.com/modify-header-value.html)
##### RestMan:
RESTMan is a browser extension to work on http requests.
[RestMan](https://chrome.google.com/webstore/detail/restman/ihgpcfpkpmdcghlnaofdmjkoemnlijdi)

#DELETE Records reamin
#How to delete records that we created.
Lets do these steps with Kubernetes:
```bash
docker-compose down
```
kompose Installation:
# Linux
```bash
curl -L https://github.com/kubernetes/kompose/releases/download/v1.25.0/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose
#OR
brew install kompose
```
# Convert time
just run :
```bash
mkdir postgres
cd postgres
cp ../docker-compose.yaml .
kompose convert
rm docker-compose.yaml
```
And Then Apply yaml files:
```bash
kubectl apply -f .
```

OR If you ran "kompose convert -o parse.yaml" then just run:
```bash
kubectl apply -f parse.yaml
```

Two way to find out why your pod not ready (if it happens):
```bash
#First one:
kubectl logs pod/server-74cd8bb94d-bhtwt   #your pod name
#Second one:
kdes pod/server-74cd8bb94d-bhtwt   #describe of your not ready pod
```
To access to your Parse:
```bash
#The CLUSTER-IP  of your service/server
curl <CLUSTER-IP>:1337/parse/health
curl 10.43.33.149:1337/parse/health
```
It returns:
```bash
{"status":"ok"}
```

Now lets expose this port via nodePort service:
```bash
mkdir nodePort
cd nodePort
cp ../postgres/*  .

```
You have to add "nodePort: 30001" & type: NodePort under spec section of server-service:
```bash
spec:
  ports:
    - name: "1337"
      protocol: "TCP"
      port: 1337
      targetPort: 1337
      nodePort: 30001
  selector:
    io.kompose.service: server
  type: NodePort
status:
  loadBalancer: {}

```
and then apply changes:
```bash
kubectl apply -f .
```
Now lets POST a sample record:
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://localhost:30001/parse/classes/GameScore
```
You should get a response similar to this:
```bash
{
"objectId":"kJx7buQPDW",
"createdAt":"2021-12-08T09:17:08.682Z"
}
Get the record (Notice the objectId at least of the url):
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:30001/parse/classes/GameScore/kJx7buQPDW  
```
Get all records that you defined:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:30001/parse/classes/GameScore
```

# Helming!
```bash
echo $PWD
#/root/ParseChart
mkdir charts
cd charts
helm create Parse
```
First, delete everything under templates directory:
```bash
rm -r Parse/templates/*
rmdir Parse/charts
```
and then copy all yaml file from convert (placed postgres) and place those files under the templates directory:
```bash
cp ../postgres/* $PWD/Parse/templates/
```
Lets install our app,First delete previus deployment:
```bash
cd ../../postgres
kubectl delete -f .
cd ../charts/Parse
helm install parse .
```
Notice: in "helm install parse ." You won't be able to set an uppercase name.

##### Values.yaml
Lets create/empty values.yaml to set our envs:

```bash
 cat <<EOF > values.yaml
server:
 - name: PARSE_SERVER_APPLICATION_ID
   value: MyParseApp
 - name: PARSE_SERVER_MASTER_KEY
   value: adminadmin 
 - name: PARSE_SERVER_DATABASE_URI
   value: postgres://postgres:postgres@postgres/postgres
EOF
```

Now we have to call values inner server-deployment.yaml:
```bash
nano templates/server-deployment.yaml
```
then modify it as below:
```bash
containers:
        - env:
            {{- range .Values.server }}
            - name: {{ .name }}
              value: {{ .value }}
            {{- end }} 
          image: parseplatform/parse-server
          name: server
          ports:
            - containerPort: 1337
          resources: {}
      restartPolicy: Always
```
save the file and run helm install contains your values file:	  
```bash
helm upgrade parse . -f values.yaml
kubectl get all -o wide 
```

# Ingress 
Let's add ingress to project:
```bash
cd charts/Parse
touch templates/ingress.yaml
nano templates/ingress.yaml
```
Paste :
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: rfinland.net
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: server
              port:
                number: 1337
```
I set "- host: rfinland.net" this is what I wrote in /etc/hosts for my  host IP:
```bash
ip -br -c a    #eth0 ip address in my case
```
then copy the host IP in my case "192.168.1.209" and then set it in /etc/hosts
192.168.1.209 rfinland.net
```
Apply changes:
```bash
helm upgrade parse . -f values.yaml
kubectl get ingress
```
```bash
#Test On your CLUSTER-IP of(service/server) first :
curl 10.43.27.73:1337/parse/health
#On our Ingress:
curl http://rfinland.net/parse/health
```

# GitHub Pages
Create a new branch for github pages:
```bash
#Create a new branch:
git checkout --orphan gh-pages
#We have to make it empty:
git rm -rf .
#Commit & Push the new branch:
git commit --allow-empty -m "root commit"
git push origin gh-pages
#Chech the current branch:
git branch
#Switch to the branch:
git checkout gh-pages
#Switch to master branch:
git checkout master
#Delete branch: When your current branch is main
git branch -d gh-pages
#Delete branch remotely
git push origin --delete gh-pages
```

Lets create an index.html file on our new branch:
```bash
git checkout gh-pages
touch index.html
echo "Hello!" > index.html
git add .
git commit -m "A index html to GitHub page"
git push --set-upstream origin gh-pages
```

And then visit:
```bash
https://rfinland.github.io/ParseChart/index.html
#https://YOURGITHUBUSER.github.io/YOURPROJECT/index.html
#OR just visit:
https://rfinland.github.io/ParseChart/
```
You should able to see what's happen in github actions on your repo
Also you can set theme for your page and add a README.md
For open source projects, GitHub Pages is a great choice to host Helm repositories. 
We’re using the gh-pages branch to store and serve the packaged charts in this part of article. 
After each release we undergo a manual process of packaging and pushing the new chart version to the gh-pages branch.