# drone-streaming-extraction
Drone API for Entity Extraction

# Architecture

- rtmp source (drone)
- rtmp backend (gcp container)
- rtmp processor/pipeline (client 1 of n)


# Experimentation around the RTMP protocol
## rtmp backend (F5 Nginx)
- https://www.nginx.com/blog/video-streaming-for-remote-learning-with-nginx/

check https://hub.docker.com/r/tiangolo/nginx-rtmp/

On i7-8700 Mac Mini (bare metal Ubuntu 22.04)
```
sudo apt install docker.io
sudo usermod -aG docker ubuntu
sudo reboot now
ubuntu@mini5:~$ docker run -d -p 1935:1935 --name nginx-rtmp tiangolo/nginx-rtmp
ubuntu@mini5:~$ docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS         PORTS                                       NAMES
cd5ebbac51a3   tiangolo/nginx-rtmp   "nginx -g 'daemon of…"   5 seconds ago   Up 2 seconds   0.0.0.0:1935->1935/tcp, :::1935->1935/tcp   nginx-rtmp

```

### Test with OBS
https://hub.docker.com/r/tiangolo/nginx-rtmp/

<img width="501" alt="Screenshot 2023-04-01 at 10 01 35" src="https://user-images.githubusercontent.com/24765473/229293737-a3b192fc-3b25-491b-ab16-6aaebfd14858.png">

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
