---
title: 'About Kubernetes configuration: ConfigMap'
date: 2022-05-08 19:38:32
tags: Kubernetes
comments: true
---
<!-- more -->

# Kubernete Configuration - ConfigMap

# Creating ConfigMap

## Create from folder

`kubectl create folder-cm --from-file <folder_name>`

```bash
# Create with folder
$ kubectl create folder-cm --from-file config
$ kubectl get cm/folder-cm -o yaml
apiVersion: v1
data:
  config.txt: |
    Test=test
  test.txt: |
    DONT=KNOW
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-02T15:53:44Z"
  name: folder-cm
  namespace: default
  resourceVersion: "41428"
  uid: 9f71a652-2350-4d55-be6e-5ef468cd45f8

```

## Create from file

💡Can create with multiple files, if create with all the files in the same folder, it’s same with creating from folder:

```bash
## Create with file
$ kubectl create cm file-cm --from-file config/config.txt --from-file config/test.txt
$ kubectl get cm/file-cm -o yaml

apiVersion: v1
data:
  config.txt: |
    Test=test
  test.txt: |
    DONT=KNOW
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-02T16:03:36Z"
  name: file-cm
  namespace: default
  resourceVersion: "42433"
  uid: 17766fc1-cfd5-4b95-b00f-296844f23407
```

The name of the file will be the key of key-value pair in `ConfigMap` while the content of the file will be the value.

If we want some specific keys, we can create the ConfigMap with given key:

```bash
$ kubectl create cm key-cm --from-file=key=config.txt
$ kubectl get cm/key-cm -o yaml
apiVersion: v1
data:
  key: |
    DB_URL=localhost:3306
    DB_USERNAME=postgres
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-03T15:16:25Z"
  name: key-cm
  namespace: default
  resourceVersion: "63721"
  uid: eb3c93ca-fcfc-42bd-82ce-2d8cddbfc900
```

## Create from key-value pairs

```bash
$ kubectl create cm value-cm --from-literal=Test=test
$ kubectl get cm/value-cm -o yaml
apiVersion: v1
data:
  Test: test
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-03T15:26:07Z"
  name: value-cm
  namespace: default
  resourceVersion: "64703"
  uid: f5431bb3-6276-48e8-85e0-33b0ef1a7243
```

## Create from env file

💡env file can be .`env` file and `.txt` file and so on ..

```bash
$ kubectl create cm env-cm --from-env-file=config.txt
$ kubectl get cm/env-cm -o yaml
apiVersion: v1
data:
  DB_URL: localhost:3306
  DB_USERNAME: postgres
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-03T15:30:49Z"
  name: env-cm
  namespace: default
  resourceVersion: "65184"
  uid: 60dba811-8fa9-4bb1-9e7b-9d45a02009eb
```

Comparing with `Create from file`, in which same file is used to create configMap, instead of using filename/given key as key, and file content as value, `creating from env file` reads the file first, and uses `key-value pairs` in the file to create ConfigMap.

# Using configMap

## Use ConfigMap as files in pod

### Mount configMap

ConfigMap can be mounted to pods as volumes. Here is the yaml file of a pod who mounts all the ComfigMaps we’ve created above (folder-cm, file-cm, value-cm, env-cm):

```yaml
# file-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: file-pod
spec:
  containers:
  - image: nginx
    name: pod
    ports:
    - containerPort: 80
    resources: {}
    volumeMounts:
    - name: folder
      mountPath: "folder"
    - name: file
      mountPath: "file"
    - name: value
      mountPath: "value"
    - name: env
      mountPath: "env"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: folder
    configMap:
      name: folder-cm
  - name: file
    configMap:
      name: file-cm
  - name: value
    configMap:
      name: value-cm
  - name: env
    configMap:
      name: env-cm
status: {}
```

Then if we log into the pod and check what is going on inside:

```bash
$ kubectl exec -it pod/file-pod -- bash
# Inside pod
$ tree -L 2
...
|-- env
|   |-- DB_URL -> ..data/DB_URL
|   `-- DB_USERNAME -> ..data/DB_USERNAME
|-- file
|   |-- config.txt -> ..data/config.txt
|   `-- test.txt -> ..data/test.txt
|-- folder
|   |-- config.txt -> ..data/config.txt
|   `-- test.txt -> ..data/test.txt
|-- value
|   `-- Test -> ..data/Test
...
```

Apparently, no matter from what we created the `ConfigMap`, the pod who mounted them would generate a file for each key-value pairs in `ConfigMap` using key as filename and value as content.

💡Mounting ConfigMap to pods as volumes is setting to `Read-Only` by default.

### Mount ConfigMap with SubPath

It’s mentioned that when we mount a `ConfigMap` to a pod, a folder named as what is specified in `spec.containers.volumeMounts.mountPath` in the yaml file will be created (if there’s already a folder with same name, the original folder will be covered and the files in this folder cannot been seen anymore.)

But there will be a situation that we only want to create a file based on the `ConfigMap` and put it into a folder which already exited. 

For example, folder `/etc/nginx/conf.d` which already has a file `default.conf` for Nginx. Instead of making this file disappear, we only want to add a new file into this folder. Then we need `SubPath`:

```yaml
# file-pod2
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: file-pod2
spec:
  containers:
  - image: nginx
    name: pod
    ports:
    - containerPort: 80
    resources: {}
    volumeMounts:
    - name: folder
      mountPath: "folder"
    - name: file
      mountPath: "/etc/nginx/conf.d/config.txt"
      subPath: "config.txt"
    - name: value
      mountPath: "value"
    - name: env
      mountPath: "env"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: folder
    configMap:
      name: folder-cm
  - name: file
    configMap:
      name: file-cm
  - name: value
    configMap:
      name: value-cm
  - name: env
    configMap:
      name: env-cm
status: {}
```

According to documentation of `Kubernetes 1.24`:

> The `volumeMounts.subPath`property specifies a sub-path inside the referenced volume instead of its root.
> 

So, consider each volume is a folder, then for `file` volume, it contains `config.txt` file and `text.txt` file. Then we can use relative path “config.txt” to point to `config.txt` file of `file volume` in `spec.containers.volumeMounts.subPath` property.

If we apply this yaml file again and log into the new pod:

```bash
$ ls
... env folder value ...
```

Other folders are still here but just `file` folder is gone. Then we can check `/etc/nginx/conf.d` folder.

```bash
$ ls /etc/nginx/conf.d
config.txt  default.conf
$ cat /etc/nginx/conf.d/config.txt
Test=test
```

`default.conf` file is still here and only `config.txt` file which is specified in `subPath` is create in this folder.

### Update ConfigMap

Now, let’s see what will happen if we try to modify a ConfigMap. We edit two ConfigMaps: `file-cm` and `value-cm`:

```yaml
#file-cm
apiVersion: v1
data:
  config.txt: |
    Test=file-modified
  test.txt: |
    DONT=KNOW
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-02T16:03:36Z"
  name: file-cm
  namespace: default
  resourceVersion: "281558"
  uid: 17766fc1-cfd5-4b95-b00f-296844f23407

#value-cm
apiVersion: v1
data:
  Test: value-modified
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-03T15:26:07Z"
  name: value-cm
  namespace: default
  resourceVersion: "317489"
  uid: f5431bb3-6276-48e8-85e0-33b0ef1a7243
```

Then we can log into file-pod2, which we created in previous section, and check what happened after we modified the `ConfigMap`:

```bash
$ kubectl exec -it pod/file-pod2 -- bash
# Content has been updated
$ cat value/Test
value-modified

# Content remains the same
$ cat /etc/nginx/conf.d/config.txt
Test=test
```

According to the result, after a `ConfigMap` is changed, the container who mounted this `ConfigMap` as volume will be updated as well. But there are several exceptions. 

One of these exceptions is that if the container uses the `ConfigMap` as `SubPath` volume, then it won’t receive the update.

💡`ConfigMap` can be set immutable, `immutable ConfigMap` will not be watched constantly, which can reduce load on kube-`apiserver`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

## Use ConfigMap as environment variables in pod

### Using specific values in ConfigMap as environment variables

In order to use specific values in ConfigMap as `environment variables` in container, we used `spec.containers.env.valueFrom.configMapKeyRef`:

```yaml
# env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: env-pod
spec:
  containers:
  - image: nginx
    name: pod
    ports:
    - containerPort: 80
    resources: {}
    env:
    - name: FOLDER_KEY
      valueFrom:
        configMapKeyRef:
          name: folder-cm
          key: test.txt
    - name: ENV_KEY
      valueFrom:
        configMapKeyRef:
          name: env-cm
          key: DB_URL
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Then create a new pod with this yaml file and checke the environment variables of it’s container:

```bash
$ kubectl exec pod/env-pod -- env
...
FOLDER_KEY=DONT=KNOW

ENV_KEY=localhost:3306
...
```

The name of the variables are what we specified with `spec.containers.env.name`. Then it will search in the target `ConfigMap` which we specified with `spec.env.valueFrom.configMapKeyRef.name` in order to find the target key-value pair whose key equals to what we specified in `spec.env.valueFrom.configMapKeyRef.key`. The value of this environment variable will be the value of target key-value pair.

### Using all values in ConfigMap as ****environment variables****

Here we will use `spec.containers.envFrom.configMapRef` to inject all the key-value pairs of a `ConfigMap` to a container as environment variables:

```yaml
# all-env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: all-env-pod
spec:
  containers:
  - image: nginx
    name: pod
    ports:
    - containerPort: 80
    resources: {}
    envFrom:
    - configMapRef:
        name: env-cm
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

After this yaml file is applied, the environment variables of the new containers are:

```yaml
$ kubectl exec pod/all-env-pod -- env
...
DB_URL=localhost:3306
DB_USERNAME=postgres
...
```

All the key-value pairs of `ConfigMap` are used as environment variable and their keys are the variable names while their values are the variable values.

### Update ConfigMap

Same as what we mentioned in previous section - `Use ConfigMap as files in pod`, we are going to modified the ConfigMap and see what will happen to corresponding containers:

```yaml
## env-cm
data:
  DB_URL: localhost:3306_modified
  DB_USERNAME: postgres
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-03T15:30:49Z"
  name: env-cm
  namespace: default
  resourceVersion: "65184"
  uid: 60dba811-8fa9-4bb1-9e7b-9d45a02009eb
```

We changed the value of `DB_URL`, then environment variables of the previous two containers are:

```bash
$ kubectl exec pod/env-pod -- env
...
FOLDER_KEY=DONT=KNOW

ENV_KEY=localhost:3306
...

$ kubectl exec pod/all-env-pod -- env
...
DB_URL=localhost:3306
DB_USERNAME=postgres
...
```

The environment variables who are created from ConfigMap remain the same. This is the other situation that we talked about before in which corresponding containers will not receive the update of the ConfigMap.