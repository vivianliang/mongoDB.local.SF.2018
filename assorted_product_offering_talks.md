# 1. Evolving data access with MongoDB stitch
Drew DiPalma - sr. product manager, mongodb

serverless layer for application to talk to database.

decrease time spend on maintenance
deploy and scale application seamlessly

4 services:
- queryanywhere. SDK for ios, android, web, iot
- functions. integrate server-side logic/microservices/cloud services
- triggers. real-time notifications based on database changes
- mobile sync. synchronize data between documents held locally in mongodb mobile and backend database. (beta)

application gets token. then can make requests. use Filters, Roles, and Rules. (example permissions, isSecret...)

field level read/write rules
schema validation

integrations. ex. twilio, s3 service, ses aws service. right now, support for: twilio, github, several aws services.

more complex stuff:
stitch functions: functions written in es6 that are scalable, hosted within stitch.

frontend hosting

configs can be exported/imported via cli

logs

# 2. General Keynote
Eliot Horowitz
CTO & Co-Founder, MongoDB
- general overview of mongodb and demos of product offerings: atlas, global config, stitch, charts

# 3. MongoDB OpsManager + Kubernetes
Norberto Leite

tldr; `MongoDB Enterprise Kubernetes Operator` allows us to ues Ops Manager to deploy mongo clusters within Kubernetes clusters.

## high level overview of kubernetes
kubernetes concepts
multi-master kubernetes with `kubeadm`
kubernetes replicaset-across nodes, single node

## ops manager / cloud manager <-- hosted version of ops manager
- tool mongo has to deploy clusters
- VSCO uses chef for this. change config of replica sets for example.
    what it does for you
        - monitoring
        - automation (ex. replica set configuration)
        - backup

## kubernetes cluster vs. mongodb cluster
there's shared terminology that mean different things. some disambiguation:

- MongoDB replica set / kubernetes replica set
    + replica sets in kubernetes operate on their own. different instances. instance level.
    + mongodb replica sets share state. always same data replicated across nodes. state level.
- MongoDB node / kubernetes node
    + kubernetes nodes: physical or virtual machine that lets us run multiple or single pods. a master node or a worker node. primary goal is to run instances.
        * application-level
    + mongo db nodes can run inside a kubernetes node

## kubernetes operators
- method of packaging deploying and maanging kubernetes application.

### ops manager kubernetes operator
MongoDB enterprise kubernetes operator (beta)
    - enables easy deploys of MongoDB into kubernetes clusters 
main benefits:
    - auto-healing using kubernetes reliability features
