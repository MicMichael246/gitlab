# GitLab Self Host Setup with Docker Compose and HAProxy and Configuring Semantic Versioning

## (1) Hosts Configuration

### Windows

##### Run the Powershell with "Run as Administrator" way:
```bash
PS C:\Users\Webhouse>
>> ipconfig
```
##### Output:
```bash
IPv4 Address. . . . . . . . . . . : 172.25.176.1
```

##### Then Add it to:
```powershell
notepad C:\Windows\System32\drivers\etc\hosts
```
##### Add these end of the file:
```bash
172.25.176.1 gitlab.example.com
172.25.176.1 registry.gitlab.example.com
```

### Linux (WSL)
```bash
nano /etc/hosts
```
##### Add these end of the file (In WSL):
```bash
172.25.176.1 gitlab.example.com
172.25.176.1 registry.gitlab.example.com
```

## (2) SSL Certificate Generation

### Directory Structure
```bash
pwd
# /root

cd gitlab
cd ssl
```

### Create SSL Configuration File
```bash
nano ssl/san.cnf
```
##### Copy this:
```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
C  = US
ST = State
L  = City
O  = ExampleCompany
CN = gitlab.example.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = gitlab.example.com
DNS.2 = registry.gitlab.example.com
```

### Generate SSL Certificate
```bash
pwd
# /root/gitlab/ssl
```

##### Generate your .crt and .key:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout gitlab.example.com.key \
  -out gitlab.example.com.crt \
  -config san.cnf
```

###### Create PEM File:
```bash
ls
# gitlab.example.com.crt  gitlab.example.com.key  san.cnf

cat gitlab.example.com.crt gitlab.example.com.key > gitlab.example.com.pem

ls
# gitlab.example.com.crt  gitlab.example.com.key  gitlab.example.com.pem  san.cnf
```

## (3) Docker Compose & HAProxy Configuration

### Docker Compose Configuration
```bash
cd ../compose
nano docker-compose.yml
```
##### Copy this:
```yaml
version: "3.8"

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: gitlab.example.com
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.example.com'
        registry_external_url 'https://registry.gitlab.example.com:5050'
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        gitlab_rails['registry_enabled'] = true
        gitlab_rails['registry_host'] = "registry.gitlab.example.com"
        gitlab_rails['registry_api_url'] = "https://registry.gitlab.example.com:5050"
        gitlab_rails['registry_issuer'] = "gitlab-issuer"
        registry['enable'] = true
        registry['registry_http_addr'] = "0.0.0.0:5050"
        registry_nginx['enable'] = false
    ports:
      - "8080:80"
      - "2222:22"
      - "5050:5050"
    volumes:
      - ../config:/etc/gitlab
      - ../logs:/var/log/gitlab
      - ../data:/var/opt/gitlab
      - ../ssl:/etc/gitlab/ssl:ro
    networks:
      - gitnet

  haproxy:
    image: haproxy:2.6
    container_name: haproxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ../haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
      - ../ssl:/usr/local/etc/haproxy/certs:ro
    networks:
      - gitnet

networks:
  gitnet:
    driver: bridge
```

### HAProxy Configuration
```bash
nano haproxy/haproxy.cfg
```
##### Copy this:
```ini
global
    log stdout format raw local0
    maxconn 2000

defaults
    log global
    mode http
    option httplog
    timeout connect 10s
    timeout client 1m
    timeout server 1m

# --- Redirect HTTP to HTTPS globally ---
frontend fe_http
    bind *:80
    mode http
    redirect scheme https code 301 if !{ ssl_fc }

# --- HTTPS frontend ---
frontend fe_https
    bind *:443 ssl crt /usr/local/etc/haproxy/certs/gitlab.example.com.pem
    mode http
    option tcplog

    acl is_gitlab hdr(host) -i gitlab.example.com
    acl is_registry hdr(host) -i registry.gitlab.example.com

    use_backend be_gitlab if is_gitlab
    use_backend be_registry if is_registry

# --- GitLab backend ---
backend be_gitlab
    mode http
    server gitlab gitlab:80 check

# --- Registry backend ---
backend be_registry
    mode http
    server registry gitlab:5050 check
```

### Start Services
```bash
docker-compose up -d
```

## (4) Gitlab Runner Installation
##### Run this inside your Local:

```bash
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

sudo chmod +x /usr/local/bin/gitlab-runner

gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner

gitlab-runner start
```


## (5) GitLab Runner Registration
##### For Runner Registration run this:

```bash
sudo gitlab-runner register \
  --url "https://gitlab.example.com/" \
  --registration-token "q7KCyLns26sMJGZi1ggH" \ #You can get this from Search bar -> Admin area -> CI/CD -> runners -> ⋮ (three dots) -> Copy the Registration token
  --description "shell-runner" \
  --executor "shell" \
  --tls-ca-file "/etc/gitlab-runner/certs/gitlab.example.com.crt"
```

##### ***Registration Details that we need to apply:***
- GitLab instance URL: `https://gitlab.example.com`
- Registration token: `q7KCyLns26sMJGZi1ggH`
- Description: `shell-runner`
- Tags: `shell-runner`
- Executor: `shell`

### ***❗Error❗***
```bash
root@Aceraspire7-Kite:~/gitlab# sudo gitlab-runner register \ --url "https://gitlab.example.com/" \ --registration-token "q7KCyLns26sMJGZi1ggH" \ --description "shell-runner" \ --executor "shell" \ --tls-ca-file "/etc/gitlab-runner/certs/gitlab.example.com.crt" Runtime platform arch=amd64 os=linux pid=410 revision=bda84871 version=18.5.0 Running in system-mode. Enter the GitLab instance URL (for example, https://gitlab.com/): [https://gitlab.example.com/]: Enter the registration token: [q7KCyLns26sMJGZi1ggH]: Enter a description for the runner: [shell-runner-3]: Enter tags for the runner (comma-separated): shell-runner-3 Enter optional maintenance note for the runner: WARNING: Support for registration tokens and runner parameters in the 'register' command has been deprecated in GitLab Runner 15.6 and will be replaced with support for authentication tokens. For more information, see https://docs.gitlab.com/ci/runners/new_creation_workflow/ ERROR: Registering runner... failed correlation_id=8ca004bb3b90464d85a001f93248f960 runner=q7KCyLns2 status=execute JSON request: execute request: couldn't execute POST against https://gitlab.example.com/api/v4/runners: Post "https://gitlab.example.com/api/v4/runners": dial tcp: lookup gitlab.example.com on 8.8.8.8:53: no such host PANIC: Failed to register the runner.
```
### ***✅Fix✅*** 
### in wsl
```bash
sudo nano /etc/hosts
```

##### Add this end of the file & save (Make sure these exists):
```bash
172.25.176.1 gitlab.example.com
172.25.176.1 registry.gitlab.example.com
```

## (6) Install Semantic releases Dependencies locally
```bash
npm init -y
npm install --save-dev semantic-release @semantic-release/gitlab @semantic-release/changelog @semantic-release/git @semantic-release/commit-analyzer @semantic-release/release-notes-generator
```

### Create .releaserc.json in your Local:
```json
{
    "branches": [
        "main", "develop"
    ],
    "repositoryUrl": "https://gitlab.example.com/mygroup/hello-go.git",
    "plugins": [
        "@semantic-release/commit-analyzer",
        "@semantic-release/release-notes-generator",
        "@semantic-release/gitlab",
        "@semantic-release/changelog",
        "@semantic-release/git"
    ],
    "preset": "conventionalcommits",
    "tagFormat": "v${version}",
    "generateNotes": {
        "preset": "conventionalcommits"
    },
    "releaseRules": [
        {
            "type": "feat",
            "release": "minor"
        },
        {
            "type": "fix",
            "release": "patch"
        },
        {
            "type": "perf",
            "release": "patch"
        },
        {
            "breaking": true,
            "release": "major"
        }
    ]
}
```

## (7) Visit
```bash
https://gitlab.example.com
```

## (8) Create Application Files For Our Gitlab Pipeline To Trigger & Create The Pipeline

### main.go
```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello from Go! 🚀")
}

func main() {
    http.HandleFunc("/", handler)
    fmt.Println("Server started on port 6060")
    http.ListenAndServe(":6060", nil)
}
```

### Dockerfile
```dockerfile
# builder stage
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go mod init example.com/hello || true
RUN go build -o app .

# final image
FROM alpine:latest
WORKDIR /root/
COPY --from=builder /src/app .
EXPOSE 6060
CMD ["./app"]
```

### ***❗Error❗***
#### ***First we create this .gitlab-ci.yml for our pipeline and this will cause error in the push stage job and then we will try to fix it:***
### .gitlab-ci.yml (!Error File!)
```yaml
variables:
  IMAGE_NAME: "registry.gitlab.example.com:5050/mygroup/hello-go"
  TAG: "latest"

  
#0-
stages:
  - build
  - push


#1-
build:
  stage: build
  tags:
    - shell-runner
  script:
    - docker build -t $IMAGE_NAME:$TAG .
    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
  only:
    - develop


#2
push:
  stage: push
  tags:
    - shell-runner
  script:
    - echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" registry.gitlab.example.com:5050 --password-stdin
    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$VERSION
    - docker push $IMAGE_NAME:$VERSION
  only:
    - develop
```

### This will be the Error for the Push job
```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:02
Gitaly correlation ID: 01K8REBWFBTV6DDZANFP57ET29
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 6d14a267 as detached HEAD (ref is main)...
Skipping Git submodules setup
Executing "step_script" stage of the job script 00:03
$ echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin
Login Succeeded
$ docker push $CI_REGISTRY_IMAGE:$TAG
The push refers to repository [registry.gitlab.example.com:5050/mygroup/hello-go]
889e6f4e61f3: Preparing
5f70bf18a086: Preparing
256f393e029f: Preparing
denied: requested access to the resource is denied
Cleaning up project directory and file based variables 00:01
ERROR: Job failed: exit status 1
```

### ***✅Fix✅***

#### (A) Now do these step by step
```bash
My Repository -> Settings -> Access tokens -> Add new token -> Enter name -> Check these scopes (api, read_api, read_repository, write_repository, read_registry, write_registry) -> Create project access token -> Copy the token
```
#### (B) Create the Variables in this location
```bash
For REGISTRY_PASSWORD:
My Repository -> Settings -> CI/CD -> Variables -> Add variable -> key = REGISTRY_PASSWORD, value = glpat-OpG66nFb0Lm4m7KduhJOW286MQp1OjUH.01.0w0d34t79

For REGISTRY_USER:
My Repository -> Settings -> CI/CD -> Variables -> Add variable -> key = REGISTRY_USER, value = root
```

#### (C) Now update the .gitlab-ci.yml this way:

### .gitlab-ci.yml
```yaml
variables:
  IMAGE_NAME: "registry.gitlab.example.com:5050/mygroup/hello-go"
  TAG: "latest"

stages:
  - build
  - release
  - push

#1-
build:
  stage: build
  tags:
    - shell-runner
  script:
    - docker build -t $IMAGE_NAME:$TAG .
    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
  only:
    - develop


#2 (Also add the release stage for semantic versioning)
release:
  stage: release
  tags:
    - shell-runner
  services:
    - docker:dind
  variables:
    DOCKER_DRIVER: overlay2
  script:
    - npm ci
    - npx semantic-release
  only:
    - develop


#3 Updated
push:
  stage: push
  tags:
    - shell-runner
  needs:
    - release
  script:
    - echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" registry.gitlab.example.com:5050 --password-stdin
    - git fetch --tags --force --prune
    - git fetch origin +refs/tags/*:refs/tags/* --force
    - git describe --tags --abbrev=0
    - VERSION=$(git tag --sort=-creatordate | head -n 1 || echo "latest")

    - echo "Detected version:" $VERSION

    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$VERSION
    - docker push $IMAGE_NAME:$VERSION
  only:
    - develop
```
### ***Now Push stage will be passed but Release stage will be Failed, we will also fix that:***
#### Push Stage
- **Status**: Passed
#### Release Stage
- **Status**: Failed!!!

### ***❗Error❗***
```bash
release Failed Started 14 hours ago by Administrator Running with gitlab-runner 18.5.0 (bda84871) on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on Aceraspire7-Kite... Getting source from Git repository 00:04 Gitaly correlation ID: 01K8V49C2X1B4YE19DYW3J5ZPV Fetching changes with git depth set to 20... Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/ Checking out 811e7162 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step_script" stage of the job script 02:43 $ npm ci npm warn deprecated semver-diff@5.0.0: Deprecated as the semver package now supports this built-in. added 370 packages, and audited 571 packages in 2m 115 packages are looking for funding run npm fund for details 4 moderate severity vulnerabilities To address all issues (including breaking changes), run: npm audit fix --force Run npm audit for details. $ npx semantic-release [9:30:04 PM] [semantic-release] › ℹ Running semantic-release version 25.0.1 [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "verifyConditions" from "@semantic-release/gitlab" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "verifyConditions" from "@semantic-release/changelog" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "verifyConditions" from "@semantic-release/git" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "analyzeCommits" from "@semantic-release/commit-analyzer" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "generateNotes" from "@semantic-release/release-notes-generator" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "prepare" from "@semantic-release/changelog" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "prepare" from "@semantic-release/git" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "publish" from "@semantic-release/gitlab" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "success" from "@semantic-release/gitlab" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "fail" from "@semantic-release/gitlab" [9:30:30 PM] [semantic-release] › ✔ Run automated release from branch main on repository https://gitlab.example.com/mygroup/hello-go.git [9:30:30 PM] [semantic-release] › ✔ Allowed to push to the Git repository [9:30:30 PM] [semantic-release] › ℹ Start step "verifyConditions" of plugin "@semantic-release/gitlab" [9:30:30 PM] [semantic-release] [@semantic-release/gitlab] › ℹ Verify GitLab authentication (https://gitlab.example.com/api/v4) [9:30:30 PM] [semantic-release] › ✘ Failed step "verifyConditions" of plugin "@semantic-release/gitlab" [9:30:30 PM] [semantic-release] › ℹ Start step "verifyConditions" of plugin "@semantic-release/changelog" [9:30:30 PM] [semantic-release] › ✔ Completed step "verifyConditions" of plugin "@semantic-release/changelog" [9:30:30 PM] [semantic-release] › ℹ Start step "verifyConditions" of plugin "@semantic-release/git" [9:30:30 PM] [semantic-release] › ✔ Completed step "verifyConditions" of plugin "@semantic-release/git" [9:30:30 PM] [semantic-release] › ✘ An error occurred while running semantic-release: RequestError: self-signed certificate; if the root CA is installed locally, try running Node.js with --use-system-ca at ClientRequest.<anonymous> (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/got/dist/source/core/index.js:901:107) at Object.onceWrapper (node:events:623:26) at ClientRequest.emit (node:events:520:35) at emitErrorEvent (node:_http_client:107:11) at TLSSocket.socketErrorListener (node:_http_client:574:5) at TLSSocket.emit (node:events:508:28) at emitErrorNT (node:internal/streams/destroy:170:8) at emitErrorCloseNT (node:internal/streams/destroy:129:3) at process.processTicksAndRejections (node:internal/process/task_queues:90:21) at TLSSocket.onConnectSecure (node:_tls_wrap:1631:34) ... 2 lines matching cause stack trace ... at ssl.onhandshakedone (node:_tls_wrap:863:12) { code: 'DEPTH_ZERO_SELF_SIGNED_CERT', input: undefined, timings: { start: 1761847230730, socket: 1761847230734, lookup: 1761847230735, connect: 1761847230738, secureConnect: undefined, upload: undefined, response: undefined, end: undefined, error: 1761847230745, abort: undefined, phases: { wait: 4, dns: 1, tcp: 3, tls: undefined, request: undefined, firstByte: undefined, download: undefined, total: 15 } }, options: { request: undefined, agent: { http: undefined, https: undefined, http2: undefined }, h2session: undefined, decompress: true, timeout: { connect: undefined, lookup: undefined, read: undefined, request: undefined, response: undefined, secureConnect: undefined, send: undefined, socket: undefined }, prefixUrl: '', body: undefined, form: undefined, json: undefined, cookieJar: undefined, ignoreInvalidCookies: [secure], searchParams: undefined, dnsLookup: undefined, dnsCache: undefined, context: {}, hooks: { init: [], beforeRequest: [], beforeError: [], beforeRedirect: [], beforeRetry: [], beforeCache: [], afterResponse: [] }, followRedirect: true, maxRedirects: 10, cache: undefined, throwHttpErrors: true, username: '', password: '', http2: [secure], allowGetBody: [secure], copyPipedHeaders: true, headers: { 'user-agent': 'got (https://github.com/sindresorhus/got)', 'private-token': '[secure]', accept: 'application/json', 'accept-encoding': 'gzip, deflate, br, zstd' }, methodRewriting: [secure], dnsLookupIpVersion: undefined, parseJson: [Function: parse], stringifyJson: [Function: stringify], retry: { limit: 2, methods: [ 'GET', 'PUT', 'HEAD', 'DELETE', 'OPTIONS', 'TRACE' ], statusCodes: [ 408, 413, 429, 500, 502, 503, 504, 521, 522, 524 ], errorCodes: [ 'ETIMEDOUT', 'ECONNRESET', 'EADDRINUSE', 'ECONNREFUSED', 'EPIPE', 'ENOTFOUND', 'ENETUNREACH', 'EAI_AGAIN' ], maxRetryAfter: undefined, calculateDelay: [Function: calculateDelay], backoffLimit: Infinity, noise: 100, enforceRetryRules: [secure] }, localAddress: undefined, method: 'GET', createConnection: undefined, cacheOptions: { shared: undefined, cacheHeuristic: undefined, immutableMinTimeToLive: undefined, ignoreCargoCult: undefined }, https: { alpnProtocols: undefined, rejectUnauthorized: undefined, checkServerIdentity: undefined, serverName: undefined, certificateAuthority: undefined, key: undefined, certificate: undefined, passphrase: undefined, pfx: undefined, ciphers: undefined, honorCipherOrder: undefined, minVersion: undefined, maxVersion: undefined, signatureAlgorithms: undefined, tlsSessionLifetime: undefined, dhparam: undefined, ecdhCurve: undefined, certificateRevocationLists: undefined, secureOptions: undefined }, encoding: undefined, resolveBodyOnly: [secure], isStream: [secure], responseType: 'text', url: URL { href: 'https://gitlab.example.com/api/v4/projects/1', origin: 'https://gitlab.example.com', protocol: 'https:', username: '', password: '', host: 'gitlab.example.com', hostname: 'gitlab.example.com', port: '', pathname: '/api/v4/projects/1', search: '', searchParams: URLSearchParams {}, hash: '' }, pagination: { transform: [Function: transform], paginate: [Function: paginate], filter: [Function: filter], shouldContinue: [Function: shouldContinue], countLimit: Infinity, backoff: 0, requestLimit: 10000, stackAllItems: [secure] }, setHost: true, maxHeaderSize: undefined, signal: undefined, enableUnixSockets: [secure], strictContentLength: [secure] }, pluginName: '@semantic-release/gitlab', [cause]: Error: self-signed certificate; if the root CA is installed locally, try running Node.js with --use-system-ca at TLSSocket.onConnectSecure (node:_tls_wrap:1631:34) at TLSSocket.emit (node:events:508:28) at TLSSocket._finishInit (node:_tls_wrap:1077:8) at ssl.onhandshakedone (node:_tls_wrap:863:12) { code: 'DEPTH_ZERO_SELF_SIGNED_CERT' } } AggregateError: RequestError: self-signed certificate; if the root CA is installed locally, try running Node.js with --use-system-ca at ClientRequest.<anonymous> (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/got/dist/source/core/index.js:901:107) at file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/lib/plugins/pipeline.js:55:13 at process.processTicksAndRejections (node:internal/process/task_queues:105:5) at async pluginsConfigAccumulator.<computed> [as verifyConditions] (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/lib/plugins/index.js:87:11) at async run (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/index.js:106:3) at async Module.default (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/index.js:278:22) at async default (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/cli.js:55:5) { errors: [ RequestError: self-signed certificate; if the root CA is installed locally, try running Node.js with --use-system-ca at ClientRequest.<anonymous> (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/got/dist/source/core/index.js:901:107) at Object.onceWrapper (node:events:623:26) at ClientRequest.emit (node:events:520:35) at emitErrorEvent (node:_http_client:107:11) at TLSSocket.socketErrorListener (node:_http_client:574:5) at TLSSocket.emit (node:events:508:28) at emitErrorNT (node:internal/streams/destroy:170:8) at emitErrorCloseNT (node:internal/streams/destroy:129:3) at process.processTicksAndRejections (node:internal/process/task_queues:90:21) at TLSSocket.onConnectSecure (node:_tls_wrap:1631:34) ... 2 lines matching cause stack trace ... at ssl.onhandshakedone (node:_tls_wrap:863:12) { code: 'DEPTH_ZERO_SELF_SIGNED_CERT', input: undefined, timings: [Object], options: { request: undefined, agent: { http: undefined, https: undefined, http2: undefined }, h2session: undefined, decompress: true, timeout: { connect: undefined, lookup: undefined, read: undefined, request: undefined, response: undefined, secureConnect: undefined, send: undefined, socket: undefined }, prefixUrl: '', body: undefined, form: undefined, json: undefined, cookieJar: undefined, ignoreInvalidCookies: [secure], searchParams: undefined, dnsLookup: undefined, dnsCache: undefined, context: {}, hooks: { init: [], beforeRequest: [], beforeError: [], beforeRedirect: [], beforeRetry: [], beforeCache: [], afterResponse: [] }, followRedirect: true, maxRedirects: 10, cache: undefined, throwHttpErrors: true, username: '', password: '', http2: [secure], allowGetBody: [secure], copyPipedHeaders: true, headers: { 'user-agent': 'got (https://github.com/sindresorhus/got)', 'private-token': '[secure]', accept: 'application/json', 'accept-encoding': 'gzip, deflate, br, zstd' }, methodRewriting: [secure], dnsLookupIpVersion: undefined, parseJson: [Function: parse], stringifyJson: [Function: stringify], retry: { limit: 2, methods: [ 'GET', 'PUT', 'HEAD', 'DELETE', 'OPTIONS', 'TRACE' ], statusCodes: [ 408, 413, 429, 500, 502, 503, 504, 521, 522, 524 ], errorCodes: [ 'ETIMEDOUT', 'ECONNRESET', 'EADDRINUSE', 'ECONNREFUSED', 'EPIPE', 'ENOTFOUND', 'ENETUNREACH', 'EAI_AGAIN' ], maxRetryAfter: undefined, calculateDelay: [Function: calculateDelay], backoffLimit: Infinity, noise: 100, enforceRetryRules: [secure] }, localAddress: undefined, method: 'GET', createConnection: undefined, cacheOptions: { shared: undefined, cacheHeuristic: undefined, immutableMinTimeToLive: undefined, ignoreCargoCult: undefined }, https: { alpnProtocols: undefined, rejectUnauthorized: undefined, checkServerIdentity: undefined, serverName: undefined, certificateAuthority: undefined, key: undefined, certificate: undefined, passphrase: undefined, pfx: undefined, ciphers: undefined, honorCipherOrder: undefined, minVersion: undefined, maxVersion: undefined, signatureAlgorithms: undefined, tlsSessionLifetime: undefined, dhparam: undefined, ecdhCurve: undefined, certificateRevocationLists: undefined, secureOptions: undefined }, encoding: undefined, resolveBodyOnly: [secure], isStream: [secure], responseType: 'text', url: URL { href: 'https://gitlab.example.com/api/v4/projects/1', origin: 'https://gitlab.example.com', protocol: 'https:', username: '', password: '', host: 'gitlab.example.com', hostname: 'gitlab.example.com', port: '', pathname: '/api/v4/projects/1', search: '', searchParams: URLSearchParams {}, hash: '' }, pagination: { transform: [Function: transform], paginate: [Function: paginate], filter: [Function: filter], shouldContinue: [Function: shouldContinue], countLimit: Infinity, backoff: 0, requestLimit: 10000, stackAllItems: [secure] }, setHost: true, maxHeaderSize: undefined, signal: undefined, enableUnixSockets: [secure], strictContentLength: [secure] }, pluginName: '@semantic-release/gitlab', [cause]: [Error] } ] } Cleaning up project directory and file based variables 00:01 ERROR: Job failed: exit status 1
```
#### ***in summary***:
```bash
[9:30:30 PM] [semantic-release] › ✘ An error occurred while running semantic-release: RequestError: self-signed certificate;
```

### ***✅Fix✅***

#### (A) Locate and Copy the SSL from my Local (.crt):
```bash
root@Aceraspire7-Kite:~/gitlab/ssl# ls

# gitlab.example.com.crt  gitlab.example.com.key  gitlab.example.com.pem  san.cnf
```

```bash
root@Aceraspire7-Kite:~/gitlab/ssl# cat gitlab.example.com.crt

-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUBk/iImHSA3nnV0Nse3DpI/EcRfkwDQYJKoZIhvcNAQEL
BQAwYjELMAkGA1UEBhMCVVMxDjAMBgNVBAgMBVN0YXRlMQ0wCwYDVQQHDARDaXR5
MRcwFQYDVQQKDA5FeGFtcGxlQ29tcGFueTEbMBkGA1UEAwwSZ2l0bGFiLmV4YW1w
bGUuY29tMB4XDTI1MTAyODA0MzkzMVoXDTI2MTAyODA0MzkzMVowYjELMAkGA1UE
BhMCVVMxDjAMBgNVBAgMBVN0YXRlMQ0wCwYDVQQHDARDaXR5MRcwFQYDVQQKDA5F
eGFtcGxlQ29tcGFueTEbMBkGA1UEAwwSZ2l0bGFiLmV4YW1wbGUuY29tMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAp0XMgGjvCVqjK7aSNmEpPOVUZIrv
qy3lFdyBdSZMxJIunkbuLqfgIFA3+IAmYUL0bI1MzTPYTzINtXA9srUsJconyZLB
tRBXkpBVaPXt0ALQrRS98ZYyIpssA/XJ7SvglBTXl6KHpHys591g6PHPRD2cXJf9
jdIucbGGzrrJC3XsWa9M7y5+jRASjZBnlSR6fMUX5xUL2wGqJacjp6Di6/3pGAW+
1ebuyduHkMKCk8CIqF/tZBw0p01UqlJ25Vva55L3RCF76AuMXbjxul75z/7vzvvm
qq+FihKSJSHdXcTd+qkhMf8kpSpEBOtM3rdEs5t7j3Akt+fw5KXKR57yHQIDAQAB
o10wWzA6BgNVHREEMzAxghJnaXRsYWIuZXhhbXBsZS5jb22CG3JlZ2lzdHJ5Lmdp
dGxhYi5leGFtcGxlLmNvbTAdBgNVHQ4EFgQUzR3xBKccNYkA1+09rod6GG1apMMw
DQYJKoZIhvcNAQELBQADggEBACJZc23r4Lmfgyk+ityKi7FBspRttJySmUqiU4zd
qWx0OPqUL9nClGVaByocx9KctGWgjvw9jQqiPgZH63CtlgJYL8b/nhU9a7j4yuoo
LZLPwRd2YFyGee5xmBenGl9QALzwMxT6nSw3qFWrwGCoL0Y0Z2tGBr5KRwNXKhjM
fmkCO5xPNr9fZ3dXuRfErfTQmxBaZuFEOUcL7eEyrbMKewPXUlzaE+i6lHHjcDem
3SWO3pHXG8Fx0Sn2IaZxYnBecqRMYo+xoebdznbuj3a0sPR+3ioRteU3MnPCLDcr
7mlwpc7/6TAyMGms0hW2b5Nx4/gF3e/EVcpV4ACFqtH8kUI=
-----END CERTIFICATE-----
```

#### (B) Create a gitlab.example.com.crt file inside my Repository in Gitlab and Paste the Content of the gitlab.example.com.crt to it:
```bash
My Repository -> Code -> Open with Web IDE -> New folder -> gitlab.example.com.crt -> "Paste the Content of the gitlab.example.com.crt"
```

#### (C) Now we need to also Update the .gitlab-ci.yml:
```yaml
variables:
  IMAGE_NAME: "registry.gitlab.example.com:5050/mygroup/hello-go"
  TAG: "latest"

  
#0-
stages:
  - build
  - release
  - push
  - cleanup


#1-
build:
  stage: build
  tags:
    - shell-runner
  script:
    - docker build -t $IMAGE_NAME:$TAG .
    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
  only:
    - develop


#2- Updated
release:
  stage: release
  tags:
    - shell-runner
  services:
    - docker:dind
  variables:
    DOCKER_DRIVER: overlay2
    NODE_OPTIONS: "--use-system-ca"
    NODE_EXTRA_CA_CERTS: "./gitlab.example.com.crt"
  script:
    - npm ci
    - npm install --save-dev conventional-changelog-conventionalcommits
    - npx semantic-release
  only:
    - develop


#3-
push:
  stage: push
  tags:
    - shell-runner
  needs:
    - release
  script:
    - echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" registry.gitlab.example.com:5050 --password-stdin
    - git fetch --tags --force --prune
    - git fetch origin +refs/tags/*:refs/tags/* --force
    - git describe --tags --abbrev=0
    - VERSION=$(git tag --sort=-creatordate | head -n 1 || echo "latest")

    - echo "Detected version:" $VERSION

    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$VERSION
    - docker push $IMAGE_NAME:$VERSION
  only:
    - develop


#4- (Also add the cleanup stage for semantic versioning to delete the older versions and keeping the last 5 new versions)
cleanup_old_releases:
  stage: cleanup
  tags:
    - shell-runner
  before_script:
    - if ! command -v git &> /dev/null; then echo "Installing git..."; apt-get update && apt-get install -y git; fi
    - git config --global user.name "GitLab CI"
    - git config --global user.email "ci-runner@gitlab.example.com"
    - git remote set-url origin https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/mygroup/hello-go.git
  script:
    - echo "Fetching all tags..."
    - git fetch --tags --force
    - echo "All tags (newest first):"
    - git tag --sort=-creatordate
    - TAGS=$(git tag --sort=-creatordate | tail -n +6)
    - |
      if [ -n "$TAGS" ]; then
        echo "Deleting old tags (remote + local):"
        for tag in $TAGS; do
          echo "Deleting tag: $tag (remote + local)"
          git push --delete origin $tag || echo "Failed to delete $tag from remote"
          git tag -d $tag || echo "Failed to delete $tag locally"
        done
      else
        echo "Less than 5 tags — nothing to delete."
      fi
    - echo "Fetching tags again after cleanup..."
    - git fetch --tags --force
    - echo "Remaining tags (newest first):"
    - git tag --sort=-creatordate
  only:
    - develop
  needs:
    - release
  when: on_success


#5- Updates the Deployment file
update_k8s_deployment:
  stage: push
  tags:
    - shell-runner
  needs:
    - push
  before_script:
    - if ! command -v git &> /dev/null; then apt-get update && apt-get install -y git; fi
    - git config --global user.name "GitLab CI"
    - git config --global user.email "ci-runner@gitlab.example.com"
    - git remote set-url origin https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/mygroup/hello-go.git
  script:
    - echo "Fetching latest tags..."
    - git fetch --tags --force
    - VERSION=$(git tag --sort=-creatordate | head -n 1)
    - echo "Latest version detected: $VERSION"

    # Update deployment.yaml with new image tag
    - echo "Updating k8s/deployment.yaml with $VERSION"
    - sed -i "s|\(image:.*hello-go:\).*|\1$VERSION|g" k8s/deployment.yaml

    # Commit and push the change
    - git add k8s/deployment.yaml
    - git commit -m "chore(ci): update deployment to $VERSION [skip ci]" || echo "No changes to commit"
    - git push origin develop
  only:
    - develop
```

#### (D) You Should Push it to "develop" branch:
##### First go to this location to enable the Protected branch for "develop" branch :
```bash
My Project -> Settings -> Repository -> Protected branches
-> Add protected branch
```

##### Add the following columns :
```bash
 ______________________________________________
| Branch  | Allowed to merge | Allowed to push |
| ------- | ---------------- | --------------- |
| develop | Maintainers      | Maintainers     |
|______________________________________________|
```

### ***Now Every stage will be Passed:***
#### Build Stage
- **Status**: Passed!!!
#### Push Stage
- **Status**: Passed!!!
#### Release Stage
- **Status**: Passed!!!
#### Cleanup Stage
- **Status**: Passed!!!

## (9) Pipeline Execution
### Now Commit and Push the Files from Web IDE for Semantic Versioning

```
My Repository -> Code -> Open with Web IDE -> Source Control (Ctrl+Shift+G)
```

### 1 - Commit and Push (feat:)
##### (A) Enter this in the Commit message:
```bash
feat: changing minor (for changing the minor version)
```

##### (B) Then:
```
Click the dropdown (˅) -> Click the "Create new branch and commit" -> Write "develop -> Enter
```
### ***Pipelines:***

- **Changing Minor Pipeline**

<img width="1920" height="1080" alt="feat changing minor pipeline" src="https://github.com/user-attachments/assets/0cf78d0c-b936-4de9-a27a-55f5c139d888" />

### (A) Build Stage
- **Status**: Passed
- **Executor**: Shell

<img width="1899" height="2158" alt="feat-build" src="https://github.com/user-attachments/assets/8bf5cc23-3ad2-4b19-bbed-ebcfaf540caf" />

### (B) Release Stage
- **Status**: Passed

<img width="1899" height="2950" alt="feat-release" src="https://github.com/user-attachments/assets/9675ac80-22cc-4ace-8d39-7a2e017cf574" />


### (C) Push Stage
- **Status**: Passed

<img width="1899" height="1498" alt="feat-push" src="https://github.com/user-attachments/assets/c0b37d2d-810f-4cf6-a769-4229ad6da9b2" />

### (D) Cleanup Stage
- **Status**: Passed

<img width="1899" height="1894" alt="feat-cleanup" src="https://github.com/user-attachments/assets/8b17fe21-0adb-41e1-bda1-1a398b3a8ff9" />

### (E) Inside Contianer Registry

<img width="1920" height="1080" alt="feat-container-registry" src="https://github.com/user-attachments/assets/36985263-a5e8-43db-ac79-3c924f12f377" />

### 2 - Commit and Push (feat:)
##### Enter this in the Commit message:
```bash
feat: changing patch (for changing the patch version)
```

### ***Pipelines:***

- **Changing Patch Pipeline**

<img width="1920" height="1080" alt="fix changing patch pipeline" src="https://github.com/user-attachments/assets/8c79e862-d72c-488c-b9b8-27aa0ffaed98" />


### (A) Build Stage
- **Status**: Passed
- **Executor**: Shell

<img width="1899" height="2158" alt="fix-build" src="https://github.com/user-attachments/assets/af9ed06b-a858-41dc-a99b-b3e0d7dfe95a" />


### (B) Release Stage
- **Status**: Passed

<img width="1899" height="2950" alt="fix-release" src="https://github.com/user-attachments/assets/37e5df6d-68f2-444c-8fc3-b36415b1f184" />

### (C) Push Stage
- **Status**: Passed

<img width="1899" height="1498" alt="fix-push" src="https://github.com/user-attachments/assets/9897a7df-4b46-42ac-b38d-f7172d4723ab" />

### (D) Cleanup Stage
- **Status**: Passed

<img width="1899" height="1894" alt="fix-cleanup" src="https://github.com/user-attachments/assets/ea9e625c-2133-4381-ae35-367c73c5645f" />

### (E) Inside Contianer Registry

<img width="1920" height="1080" alt="fix-container-registry" src="https://github.com/user-attachments/assets/bc510896-88b8-4e55-b510-b6527031662f" />

## (10) Deleting Old Tags

- **Set these settings**

<img width="1899" height="1366" alt="Deleting Old Tags" src="https://github.com/user-attachments/assets/44e51b49-9f6a-40fc-8ea1-90858e685f2c" />


- **After applying settings**

<img width="1920" height="1080" alt="Deleting Old Tags -2" src="https://github.com/user-attachments/assets/8421e81d-0649-441a-96c1-713b91ddaffd" />


## (11) Create a repository inside Gitlab repository and create deployment & service files

- **k8s**

deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-go
  labels:
    app: hello-go
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-go
  template:
    metadata:
      labels:
        app: hello-go
    spec:
      containers:
      - name: hello-go
        image: registry.gitlab.example.com:5050/mygroup/hello-go:v4.0.0
        ports:
        - containerPort: 6060
      imagePullSecrets:
      - name: gitlab-registry   # <-- make sure this secret exists
```


<img width="1920" height="1080" alt="2025-11-10 23 19 56" src="https://github.com/user-attachments/assets/a63e9a4c-1497-4ada-abad-fec47614ea73" />


service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-go-service
spec:
  selector:
    app: hello-go
  ports:
  - port: 6060
    targetPort: 6060
  type: NodePort
```

## (12) Creating Kind Cluster
```bash
root@Aceraspire7-Kite:~# kind create cluster --name kind-cluster --config kind-config.yaml --image kindest/node:v1.29.2
Creating cluster "kind-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.29.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-kind-cluster

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

```bash
root@Aceraspire7-Kite:~# kubectl get nodes
NAME                         STATUS   ROLES           AGE     VERSION
kind-cluster-control-plane   Ready    control-plane   4m44s   v1.29.2
```

## (13) Create Token in Gitlab We will use this
```bash
argo-pat-1:
read_api, read_repository, write_repository


Copy the Token We will use it
```


<img width="1920" height="1080" alt="2025-11-10 23 06 29" src="https://github.com/user-attachments/assets/48e70e1a-f295-4090-89d4-afadd857bcfa" />



## (14) Installing & Creating ArgoCD Pod inside argocd namespace & other stuffs

```bash
root@Aceraspire7-Kite:~# curl -O https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 1434k  100 1434k    0     0   164k      0  0:00:08  0:00:08 --:--:--  303k


root@Aceraspire7-Kite:~# kubectl get pods -n argocd
No resources found in argocd namespace.

root@Aceraspire7-Kite:~# kubectl create namespace argocd


root@Aceraspire7-Kite:~# kubectl apply -n argocd -f install.yaml
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-applicationset-controller created
serviceaccount/argocd-dex-server created
serviceaccount/argocd-notifications-controller created
serviceaccount/argocd-redis created
serviceaccount/argocd-repo-server created
serviceaccount/argocd-server created
role.rbac.authorization.k8s.io/argocd-application-controller created
role.rbac.authorization.k8s.io/argocd-applicationset-controller created
role.rbac.authorization.k8s.io/argocd-dex-server created
role.rbac.authorization.k8s.io/argocd-notifications-controller created
role.rbac.authorization.k8s.io/argocd-redis created
role.rbac.authorization.k8s.io/argocd-server created
clusterrole.rbac.authorization.k8s.io/argocd-application-controller created
clusterrole.rbac.authorization.k8s.io/argocd-applicationset-controller created
clusterrole.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
rolebinding.rbac.authorization.k8s.io/argocd-dex-server created
rolebinding.rbac.authorization.k8s.io/argocd-notifications-controller created
rolebinding.rbac.authorization.k8s.io/argocd-redis created
rolebinding.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-server created
configmap/argocd-cm created
configmap/argocd-cmd-params-cm created
configmap/argocd-gpg-keys-cm created
configmap/argocd-notifications-cm created
configmap/argocd-rbac-cm created
configmap/argocd-ssh-known-hosts-cm created
configmap/argocd-tls-certs-cm created
secret/argocd-notifications-secret created
secret/argocd-secret created
service/argocd-applicationset-controller created
service/argocd-dex-server created
service/argocd-metrics created
service/argocd-notifications-controller-metrics created
service/argocd-redis created
service/argocd-repo-server created
service/argocd-server created
service/argocd-server-metrics created
deployment.apps/argocd-applicationset-controller created
deployment.apps/argocd-dex-server created
deployment.apps/argocd-notifications-controller created
deployment.apps/argocd-redis created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
statefulset.apps/argocd-application-controller created
networkpolicy.networking.k8s.io/argocd-application-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-applicationset-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-dex-server-network-policy created
networkpolicy.networking.k8s.io/argocd-notifications-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-redis-network-policy created
networkpolicy.networking.k8s.io/argocd-repo-server-network-policy created
networkpolicy.networking.k8s.io/argocd-server-network-policy created


root@Aceraspire7-Kite:~# kubectl get pods -n argocd
NAME                                                READY   STATUS              RESTARTS   AGE
argocd-application-controller-0                     0/1     ContainerCreating   0          5m27s
argocd-applicationset-controller-6bdfdfc75f-wxb5f   0/1     ImagePullBackOff    0          6m
argocd-dex-server-7bf9ff885c-jjg5z                  0/1     PodInitializing     0          5m59s
argocd-notifications-controller-6d565dc8d4-m9xsz    1/1     Running             0          5m53s
argocd-redis-585584c457-t4kdt                       0/1     Init:0/1            0          5m49s
argocd-repo-server-685b5b8544-lbbmk                 0/1     Init:0/1            0          5m40s
argocd-server-b54c954f9-65qq8


root@Aceraspire7-Kite:~# kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          14m
argocd-applicationset-controller-6bdfdfc75f-wxb5f   1/1     Running   0          14m
argocd-dex-server-7bf9ff885c-jjg5z                  1/1     Running   0          14m
argocd-notifications-controller-6d565dc8d4-m9xsz    1/1     Running   0          14m
argocd-redis-585584c457-t4kdt                       1/1     Running   0          14m
argocd-repo-server-685b5b8544-lbbmk                 1/1     Running   0          14m
argocd-server-b54c954f9-65qq8                       1/1     Running   0          14m


------------------------------------------------------------------


Create a ConfigMap for your GitLab certificate:

kubectl -n argocd create configmap gitlab-cert \
  --from-file=gitlab.example.com.crt=/root/gitlab/ssl/gitlab.example.com.crt


Patch the argocd-repo-server deployment to mount it:

(A)- Add the volume mount to the container:
kubectl -n argocd patch deployment argocd-repo-server \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/volumeMounts/-", "value":{"name":"gitlab-cert","mountPath":"/etc/ssl/certs/gitlab.example.com.crt","subPath":"gitlab.example.com.crt"}}]'


(B)- Add the volume to the pod spec:
kubectl -n argocd patch deployment argocd-repo-server \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/volumes/-", "value":{"name":"gitlab-cert","configMap":{"name":"gitlab-cert"}}}]'


(C)- Restart the repo server:
kubectl -n argocd rollout restart deployment argocd-repo-server


------------------------------------------------------------------


root@Aceraspire7-Kite:~# kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.96.214.179   <none>        7000/TCP,8080/TCP            15m
argocd-dex-server                         ClusterIP   10.96.216.75    <none>        5556/TCP,5557/TCP,5558/TCP   15m
argocd-metrics                            ClusterIP   10.96.106.231   <none>        8082/TCP                     15m
argocd-notifications-controller-metrics   ClusterIP   10.96.244.37    <none>        9001/TCP                     15m
argocd-redis                              ClusterIP   10.96.164.79    <none>        6379/TCP                     15m
argocd-repo-server                        ClusterIP   10.96.48.15     <none>        8081/TCP,8084/TCP            14m
argocd-server                             ClusterIP   10.96.252.217   <none>        80/TCP,443/TCP               14m
argocd-server-metrics                     ClusterIP   10.96.135.49    <none>        8083/TCP                     14m
```


<img width="1899" height="5548" alt="2025-11-10 22 59 56" src="https://github.com/user-attachments/assets/8f858b12-b13f-4f81-9b39-8f2ab74a5c34" />



## (15) Gitlab-CLI-Installation & Adding Repository to ArgoCD
```bash
root@Aceraspire7-Kite:~/gitlab/ssl# curl -v -L -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
 99  203M   99  203M    0     0   174k      0  0:19:56  0:19:55  0:00:01  135k* TLSv1.2 (IN), TLS header, Supplemental data (23):
{ [5 bytes data]
* TLSv1.2 (IN), TLS header, Supplemental data (23):
{ [5 bytes data]
* TLSv1.2 (IN), TLS header, Supplemental data (23):
{ [5 bytes data]
* TLSv1.2 (IN), TLS header, Supplemental data (23):
{ [5 bytes data]
* TLSv1.2 (IN), TLS header, Supplemental data (23):
{ [5 bytes data]
* TLSv1.2 (IN), TLS header, Supplemental data (23):
{ [5 bytes data]
* TLSv1.2 (IN), TLS header, Supplemental data (23):
{ [5 bytes data]
100  203M  100  203M    0     0   174k      0  0:19:56  0:19:56 --:--:--  135k
* Connection #1 to host release-assets.githubusercontent.com left intact


root@Aceraspire7-Kite:~/gitlab/ssl# chmod +x argocd


root@Aceraspire7-Kite:~/gitlab/ssl# sudo mv argocd /usr/local/bin/


root@Aceraspire7-Kite:~/gitlab/ssl# argocd version
argocd: v3.2.0+66b2f30
  BuildDate: 2025-11-04T15:21:01Z
  GitCommit: 66b2f302d91a42cc151808da0eec0846bbe1062c
  GitTreeState: clean
  GoVersion: go1.25.0
  Compiler: gc
  Platform: linux/amd64
{"level":"fatal","msg":"Argo CD server address unspecified","time":"2025-11-10T20:57:50+03:30"}


------------------------------------------------------------------


kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo


root@Aceraspire7-Kite:~/gitlab/ssl# argocd login localhost:9090 --username admin --password <ArgoCD-Password> --insecure
'admin:login' logged in successfully
Context 'localhost:9090' updated


------------------------------------------------------------------


root@Aceraspire7-Kite:~# argocd repo add https://gitlab.example.com/mygroup/hello-go.git   --username root   --password <Access-Token>   --name gitlab-hello-go
Repository 'https://gitlab.example.com/mygroup/hello-go.git' added


root@Aceraspire7-Kite:~# argocd repo list
TYPE  NAME             REPO                                             INSECURE  OCI    LFS    CREDS  STATUS      MESSAGE  PROJECT
git   gitlab-hello-go  https://gitlab.example.com/mygroup/hello-go.git  false     false  false  false  Successful


------------------------------------------------------------------


***❗Error❗***

root@Aceraspire7-Kite:~# argocd app create hello-go \
  --repo https://gitlab.example.com/mygroup/hello-go.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace myapp \
  --sync-policy automated \
  --self-heal
{"level":"fatal","msg":"rpc error: code = InvalidArgument desc = application spec for hello-go is invalid: InvalidSpecError: Unable to generate manifests in k8s: rpc error: code = Unknown desc = k8s: app path does not exist","time":"2025-11-10T21:50:36+03:30"}


***✅Fix✅***

root@Aceraspire7-Kite:~# argocd app create hello-go \
  --repo https://gitlab.example.com/mygroup/hello-go.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace myapp \
  --sync-policy automated \
  --self-heal \
  --revision develop
application 'hello-go' created


------------------------------------------------------------------


root@Aceraspire7-Kite:~# argocd app list
NAME             CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH       SYNCPOLICY  CONDITIONS  REPO                                             PATH  TARGET
argocd/hello-go  https://kubernetes.default.svc  myapp      default  Synced  Progressing  Auto        <none>      https://gitlab.example.com/mygroup/hello-go.git  k8s   develop
```

<img width="1899" height="4119" alt="2025-11-10 22 58 58" src="https://github.com/user-attachments/assets/6e0eb291-a25e-4b9c-b8c5-4b9268140c23" />


## (16) Visiting ArgoCD UI
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo


root@Aceraspire7-Kite:~# kubectl port-forward -n argocd svc/argocd-server 9090:443
Forwarding from 127.0.0.1:9090 -> 8080
Forwarding from [::1]:9090 -> 8080
Handling connection for 9090
Handling connection for 9090
```
<img width="1920" height="1080" alt="2025-11-10 23 19 44" src="https://github.com/user-attachments/assets/d6e9a421-d71c-4fed-8edb-202ab063c94b" />

<img width="1920" height="1080" alt="2025-11-10 22 38 53" src="https://github.com/user-attachments/assets/11539820-0c12-4c03-b2fc-2d13dedef95b" />

<img width="1920" height="1080" alt="2025-11-10 22 38 45" src="https://github.com/user-attachments/assets/7024cac1-9b2f-4d33-bf0d-62843e12da0a" />


## (16) Checking the Pod Deployment
```bash
root@Aceraspire7-Kite:~# kubectl get pods -n myapp
NAME                        READY   STATUS   RESTARTS   AGE
hello-go-56787d8758-r9zrv   1/1     Running  0          23m
hello-go-5b8fc978dc-4g29b   1/1     Running  0          8m33s
```

<img width="874" height="154" alt="2025-11-11 20 30 14" src="https://github.com/user-attachments/assets/b60138e6-7a0c-4b3a-b44b-9d0e9f9879b6" />
