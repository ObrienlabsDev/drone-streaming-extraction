# drone-streaming-extraction
- see https://github.com/CloudLandingZone/poc-gcp-serverless-drone-stream-ai
- Drone API for Entity Extraction

# Architecture

- rtmp source (drone)
- rtmp backend (gcp container)
- rtmp processor/pipeline (client 1 of n)


# Experimentation around the RTMP protocol
## Test Locally on docker inside on-prem network

### Docker environment in VM or Bare Metal
- Refer to https://www.nginx.com/blog/video-streaming-for-remote-learning-with-nginx/
- use prepagaged container from tiangelo at https://hub.docker.com/r/tiangolo/nginx-rtmp/

### rtmp backend (F5 Nginx)
- check https://hub.docker.com/r/tiangolo/nginx-rtmp/
- On i7-8700 Mac Mini (bare metal Ubuntu 22.04)

#### RTMP on Bare Metal
```
sudo apt install docker.io
sudo usermod -aG docker ubuntu
sudo reboot now
ubuntu@mini5:~$ docker run -d -p 1935:1935 --name nginx-rtmp tiangolo/nginx-rtmp
ubuntu@mini5:~$ docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS         PORTS                                       NAMES
cd5ebbac51a3   tiangolo/nginx-rtmp   "nginx -g 'daemon of…"   5 seconds ago   Up 2 seconds   0.0.0.0:1935->1935/tcp, :::1935->1935/tcp   nginx-rtmp


logs locally
192.168.0.24 [01/Apr/2023:14:24:12 +0000] PLAY "live" "drone" "" - 417 13153219 "" "LNX 9,0,124,2" (19s)
192.168.0.19 [01/Apr/2023:14:24:52 +0000] PUBLISH "live" "drone" "" - 372055726 529 "" "" (11m 47s)

logs remote drone up the hill via cell hotspot
2023/04/01 21:15:46 [error] 9#9: *234 live: already publishing, client: , server: 0.0.0.0:1935

switched hotspot from iphone to ipad
24.114.94.29 [01/Apr/2023:21:17:38 +0000] PLAY "live" "drone" "" - 376 454 "" "LNX 9,0,124,2" (37s)
24.114.94.29 [01/Apr/2023:21:18:44 +0000] PLAY "live" "drone" "" - 376 454 "" "LNX 9,0,124,2" (1m 4s)
24.114.98.182 [01/Apr/2023:21:18:56 +0000] PUBLISH "live" "drone" "" - 30184047 412 "" "" (3m 10s)
2023/04/01 21:19:07 [error] 8#8: *256 live: already publishing, client: 24.114.98.182, server: 0.0.0.0:1935

24.114.85.203 [01/Apr/2023:21:30:52 +0000] PUBLISH "live" "drone" "" - 148641484 607 "" "" (25m 22s)
24.114.98.182 [01/Apr/2023:21:33:32 +0000] PUBLISH "live" "drone" "" - 52357335 430 "" "" (5m 0s)
2023/04/01 21:33:33 [error] 8#8: *326 live: already publishing, client: 24.114.98.182, server: 0.0.0.0:1935

24.114.98.182 [01/Apr/2023:21:35:00 +0000] PUBLISH "live" "drone" "" - 39086027 427 "" "" (3m 21s)
24.114.98.182 [01/Apr/2023:21:37:50 +0000] PUBLISH "live" "drone" "" - 6669702 448 "" "" (4m 17s)
24.114.98.182 [01/Apr/2023:21:39:30 +0000] PUBLISH "live" "drone" "" - 167359073 538 "" "" (20m 23s)

24.114.94.29 [01/Apr/2023:21:39:53 +0000] PLAY "live" "drone" "" - 492 18003597 "" "LNX 9,0,124,2" (11m 28s)

```

### Google Cloud Run

- move the dockerhub image to artifact registry
```
docker pull tiangolo/nginx-rtmp
gcloud artifacts repositories create $AR_PIPELINE_NAME --location=$REGION --repository-format=docker
check repo
gcloud artifacts repositories describe rtmp-pipeline-csr --project=drone-ol --location=$REGION

northamerica-northeast1-docker.pkg.dev/drone-ol/rtmp-pipeline-csr

add permissions as per helper
gcloud auth configure-docker northamerica-northeast1-docker.pkg.dev
    
add artifiact registry administrator
docker tag tiangolo/nginx-rtmp:latest ${REGION}-docker.pkg.dev/drone-ol/$AR_PIPELINE_NAME/rtmp-pipeline-csr:latest
docker push ${REGION}-docker.pkg.dev/drone-ol/$AR_PIPELINE_NAME/$AR_PIPELINE_NAME:latest

```
<img width="1485" alt="Screenshot 2023-04-02 at 13 58 37" src="https://user-images.githubusercontent.com/24765473/229370411-1967da4a-1de4-449e-a08c-2bcc50aabd14.png">



https://nginx-rtmp-kkxj4lcnsa-uc-old.a.run.app

```
gcloud run deploy nginx-rtmp \
--image=gcr.io/drone-ol/rtmp-pipeline-csr@sha256:e349d276df7319b668c3155cd4ef5255cd1fc0c520dc119bd7dea004580a22fc \
--allow-unauthenticated \
--port=1935 \
--service-account=452219143276-compute@developer.gserviceaccount.com \
--cpu=2 \
--memory=1Gi \
--min-instances=1 \
--max-instances=4 \
--no-cpu-throttling \
--execution-environment=gen2 \
--region=us-central1 \
--project=drone-ol

```

### Google Cloud Compute Engine

add firewall rules
```
gcloud compute --project=drone-ol firewall-rules create rtmp --description=rtmp --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:1935 --source-ranges=0.0.0.0/0
gcloud compute --project=drone-ol firewall-rules create rtmp-out --description="rtmp out" --direction=EGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:1935 --destination-ranges=0.0.0.0/0
```

```
gcloud compute instances create-with-container rtmp --project=drone-ol --zone=northamerica-northeast1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=452219143276-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --tags=http-server,https-server --image=projects/cos-cloud/global/images/cos-stable-101-17162-127-51 --boot-disk-size=10GB --boot-disk-type=pd-balanced --boot-disk-device-name=rtmp --container-image=gcr.io/drone-ol/rtmp-pipeline-csr@sha256:e349d276df7319b668c3155cd4ef5255cd1fc0c520dc119bd7dea004580a22fc --container-restart-policy=always --container-privileged --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --labels=ec-src=vm_add-gcloud,container-vm=cos-stable-101-17162-127-51
```

### Test with OBS
https://hub.docker.com/r/tiangolo/nginx-rtmp/

<img width="501" alt="Screenshot 2023-04-01 at 10 01 35" src="https://user-images.githubusercontent.com/24765473/229293737-a3b192fc-3b25-491b-ab16-6aaebfd14858.png">

### Test view with VLC

- install http://get.videolan.org/vlc/3.0.18/macosx/vlc-3.0.18-arm64.dmg
- configure
<img width="505" alt="Screenshot 2023-04-01 at 10 01 21" src="https://user-images.githubusercontent.com/24765473/229322646-9a0198b5-52f8-4770-9f19-b8364f75ade2.png">

- run
<img width="1138" alt="Screenshot 2023-04-01 at 19 26 45" src="https://user-images.githubusercontent.com/24765473/229322635-b937b4a5-5520-4c27-beca-0118fd6af400.png">

## rtmp processor (gcp pipeline)
- https://codelabs.developers.google.com/mediacdn-ls-codelab#0
- https://cloud.google.com/livestream/docs/overview

- rtmp server to processor  https://cloud.google.com/video-intelligence/docs/streaming/live-streaming
- https://cloud.google.com/video-intelligence/docs/streaming/docker-kubernetes
- https://github.com/google/aistreamer/tree/master

in GCP Shell
```
export DOCKER_IMAGE=gcr.io/drone-ol/drone-ol:0.0.1
git clone https://github.com/google/aistreamer.git
cd aistreamer/ingestion/
docker build -t $DOCKER_IMAGE -f env/Dockerfile .
  
4 min
------
 > [ 4/28] RUN easy_install pip:
#0 0.492 Searching for pip
#0 0.492 Reading https://pypi.python.org/simple/pip/
#0 0.630 Scanning index of all packages (this may take a while)
#0 0.630 Reading https://pypi.python.org/simple/
#0 0.630 Couldn't find index page for 'pip' (maybe misspelled?)
#0 0.690 No local packages or download links found for pip
#0 0.691 error: Could not find suitable distribution for Requirement.parse('pip')
------
Dockerfile:80
--------------------
  78 |         && apt-get clean
  79 |
  80 | >>> RUN easy_install pip
  81 |
  82 |     RUN add-apt-repository ppa:jonathonf/ffmpeg-4 -y
--------------------
ERROR: failed to solve: process "/bin/sh -c easy_install pip" did not complete successfully: exit code: 1
```
