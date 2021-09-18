# Using Joint Inference Service in Helmet Detection Scenario on S3

This example is based on the example: [Using Joint Inference Service in Helmet Detection Scenario](/examples/joint_inference/helmet_detection_inference/README.md).  

### Prepare Nodes
Assume you have created a [KubeEdge](https://github.com/kubeedge/kubeedge) cluster that have one cloud node(e.g., `cloud-node`)
and one edge node(e.g., `edge-node`).  

### Create a secret with your S3 user credential.        

```shell
kubectl create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  annotations:
    s3-endpoint: play.min.io # replace with your s3 endpoint
    s3-usehttps: "1" # by default 1, if testing with minio you can set to 0
stringData:
  ACCESS_KEY_ID: Q3AM3UQ867SPQQA43P2F
  SECRET_ACCESS_KEY: zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG
EOF
```

### Prepare Model  
* Download [little model](https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection-inference/little-model.tar.gz)
  and [big model](https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection-inference/big-model.tar.gz).  
* Put the unzipped model file into the bucket of your cloud storage service.  
* Attach the created secret to the Model and create Model.  

```shell
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Model
metadata:
  name: big-model
  spec:
    url : "s3://kubeedge/model/big-model/yolov3_darknet.pb"
    format: "pb"
    credentialName: mysecret
EOF
```

```shell
kubectl $action -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Model
metadata:
  name: little-model
spec:
  url: "s3://kubeedge/model/little-model/yolov3_resnet18.pb"
  format: "pb"
  credentialName: mysecret
EOF
```

### Prepare Images
This example uses these images:  
1. little model inference worker: ```kubeedge/sedna-example-joint-inference-helmet-detection-little:v0.3.0```
2. big model inference worker: ```kubeedge/sedna-example-joint-inference-helmet-detection-big:v0.3.0```

These images are generated by the script [build_images.sh](/examples/build_image.sh).  

### Prepare Job 
* Make preparation in edge node  

```
mkdir -p /joint_inference/output
```

* Attach the created secret to the Job and create Job.    

```shell
LITTLE_MODEL_IMAGE=kubeedge/sedna-example-joint-inference-helmet-detection-little:v0.3.0
BIG_MODEL_IMAGE=kubeedge/sedna-example-joint-inference-helmet-detection-big:v0.3.0

kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: JointInferenceService
metadata:
  name: helmet-detection-inference-example
  namespace: default
spec:
  edgeWorker:
    model:
      name: "helmet-detection-inference-little-model"
    hardExampleMining:
      name: "IBT"
      parameters:
        - key: "threshold_img"
          value: "0.9"
        - key: "threshold_box"
          value: "0.9"
    template:
      spec:
        nodeName: edge-node
        containers:
          - image: $LITTLE_MODEL_IMAGE
            imagePullPolicy: IfNotPresent
            name:  little-model
            env:  # user defined environments
              - name: input_shape
                value: "416,736"
              - name: "video_url"
                value: "rtsp://localhost/video"
              - name: "all_examples_inference_output"
                value: "/data/output"
              - name: "hard_example_cloud_inference_output"
                value: "/data/hard_example_cloud_inference_output"
              - name: "hard_example_edge_inference_output"
                value: "/data/hard_example_edge_inference_output"
            resources:  # user defined resources
              requests:
                memory: 64M
                cpu: 100m
              limits:
                memory: 2Gi
            volumeMounts:
              - name: outputdir
                mountPath: /data/
        volumes:   # user defined volumes
          - name: outputdir
            hostPath:
              # user must create the directory in host
              path: /joint_inference/output
              type: DirectoryorCreate

  cloudWorker:
    model:
      name: "helmet-detection-inference-big-model"
    template:
      spec:
        nodeName: cloud-node
        containers:
          - image: $BIG_MODEL_IMAGE
            name:  big-model
            imagePullPolicy: IfNotPresent
            env:  # user defined environments
              - name: "input_shape"
                value: "544,544"
            resources:  # user defined resources
              requests:
                memory: 2Gi
EOF
```

### Mock Video Stream for Inference in Edge Side
Refer to [here](https://github.com/kubeedge/sedna/tree/main/examples/joint_inference/helmet_detection_inference#mock-video-stream-for-inference-in-edge-side).