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

### ***!Error!***
```bash
root@Aceraspire7-Kite:~/gitlab# sudo gitlab-runner register \ --url "https://gitlab.example.com/" \ --registration-token "q7KCyLns26sMJGZi1ggH" \ --description "shell-runner" \ --executor "shell" \ --tls-ca-file "/etc/gitlab-runner/certs/gitlab.example.com.crt" Runtime platform arch=amd64 os=linux pid=410 revision=bda84871 version=18.5.0 Running in system-mode. Enter the GitLab instance URL (for example, https://gitlab.com/): [https://gitlab.example.com/]: Enter the registration token: [q7KCyLns26sMJGZi1ggH]: Enter a description for the runner: [shell-runner-3]: Enter tags for the runner (comma-separated): shell-runner-3 Enter optional maintenance note for the runner: WARNING: Support for registration tokens and runner parameters in the 'register' command has been deprecated in GitLab Runner 15.6 and will be replaced with support for authentication tokens. For more information, see https://docs.gitlab.com/ci/runners/new_creation_workflow/ ERROR: Registering runner... failed correlation_id=8ca004bb3b90464d85a001f93248f960 runner=q7KCyLns2 status=execute JSON request: execute request: couldn't execute POST against https://gitlab.example.com/api/v4/runners: Post "https://gitlab.example.com/api/v4/runners": dial tcp: lookup gitlab.example.com on 8.8.8.8:53: no such host PANIC: Failed to register the runner.
```
### ***~Fix~*** 
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
import "fmt"
func main(){ fmt.Println("Hello from Go!") }
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
CMD ["./app"]
```

### ***!Error!***
#### ***First we create this .gitlab-ci.yml for our pipeline and this will cause error in the push stage job and then we will try to fix it:***
### .gitlab-ci.yml (!Error File!)
```yaml
variables:
  TAG: "latest"
  
stages:
  - build
  - push

build:
  stage: build
  tags:
    - shell-runner
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$TAG .
    - docker tag $CI_REGISTRY_IMAGE:$TAG $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

  only:
    - develop    


push:
  stage: push
  tags:
    - shell-runner
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" $CI_REGISTRY --password-stdin
    - docker push $CI_REGISTRY_IMAGE:$TAG
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

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

### ***~Fix~***

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
  - push
  - release
#1
build:
  stage: build
  tags:
    - shell-runner
  script:

    - docker build -t $IMAGE_NAME:$TAG .
    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$CI_COMMIT_SHORT_SHA

  only:
    - develop
#2 Updated
push:
  stage: push
  tags:
    - shell-runner
  script:
    - echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" registry.gitlab.example.com:5050 --password-stdin
    - VERSION=$(git describe --tags --abbrev=0 || echo "latest")
    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$VERSION
    - docker push $IMAGE_NAME:$TAG
    - docker push $IMAGE_NAME:$VERSION
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA

    - docker pull $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker pull $IMAGE_NAME:$VERSION

  only:
    - develop
#3 (Also add the release stage for semantic versioning)
release:
  stage: release
  image: node:24-bullseye
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
```
### ***Now Push stage will be passed but Release stage will be Failed, we will also fix that:***
#### Push Stage
- **Status**: Passed
#### Release Stage
- **Status**: Failed!!!

### ***!Error!***
```bash
release Failed Started 14 hours ago by Administrator Running with gitlab-runner 18.5.0 (bda84871) on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on Aceraspire7-Kite... Getting source from Git repository 00:04 Gitaly correlation ID: 01K8V49C2X1B4YE19DYW3J5ZPV Fetching changes with git depth set to 20... Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/ Checking out 811e7162 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step_script" stage of the job script 02:43 $ npm ci npm warn deprecated semver-diff@5.0.0: Deprecated as the semver package now supports this built-in. added 370 packages, and audited 571 packages in 2m 115 packages are looking for funding run npm fund for details 4 moderate severity vulnerabilities To address all issues (including breaking changes), run: npm audit fix --force Run npm audit for details. $ npx semantic-release [9:30:04 PM] [semantic-release] › ℹ Running semantic-release version 25.0.1 [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "verifyConditions" from "@semantic-release/gitlab" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "verifyConditions" from "@semantic-release/changelog" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "verifyConditions" from "@semantic-release/git" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "analyzeCommits" from "@semantic-release/commit-analyzer" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "generateNotes" from "@semantic-release/release-notes-generator" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "prepare" from "@semantic-release/changelog" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "prepare" from "@semantic-release/git" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "publish" from "@semantic-release/gitlab" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "success" from "@semantic-release/gitlab" [9:30:22 PM] [semantic-release] › ✔ Loaded plugin "fail" from "@semantic-release/gitlab" [9:30:30 PM] [semantic-release] › ✔ Run automated release from branch main on repository https://gitlab.example.com/mygroup/hello-go.git [9:30:30 PM] [semantic-release] › ✔ Allowed to push to the Git repository [9:30:30 PM] [semantic-release] › ℹ Start step "verifyConditions" of plugin "@semantic-release/gitlab" [9:30:30 PM] [semantic-release] [@semantic-release/gitlab] › ℹ Verify GitLab authentication (https://gitlab.example.com/api/v4) [9:30:30 PM] [semantic-release] › ✘ Failed step "verifyConditions" of plugin "@semantic-release/gitlab" [9:30:30 PM] [semantic-release] › ℹ Start step "verifyConditions" of plugin "@semantic-release/changelog" [9:30:30 PM] [semantic-release] › ✔ Completed step "verifyConditions" of plugin "@semantic-release/changelog" [9:30:30 PM] [semantic-release] › ℹ Start step "verifyConditions" of plugin "@semantic-release/git" [9:30:30 PM] [semantic-release] › ✔ Completed step "verifyConditions" of plugin "@semantic-release/git" [9:30:30 PM] [semantic-release] › ✘ An error occurred while running semantic-release: RequestError: self-signed certificate; if the root CA is installed locally, try running Node.js with --use-system-ca at ClientRequest.<anonymous> (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/got/dist/source/core/index.js:901:107) at Object.onceWrapper (node:events:623:26) at ClientRequest.emit (node:events:520:35) at emitErrorEvent (node:_http_client:107:11) at TLSSocket.socketErrorListener (node:_http_client:574:5) at TLSSocket.emit (node:events:508:28) at emitErrorNT (node:internal/streams/destroy:170:8) at emitErrorCloseNT (node:internal/streams/destroy:129:3) at process.processTicksAndRejections (node:internal/process/task_queues:90:21) at TLSSocket.onConnectSecure (node:_tls_wrap:1631:34) ... 2 lines matching cause stack trace ... at ssl.onhandshakedone (node:_tls_wrap:863:12) { code: 'DEPTH_ZERO_SELF_SIGNED_CERT', input: undefined, timings: { start: 1761847230730, socket: 1761847230734, lookup: 1761847230735, connect: 1761847230738, secureConnect: undefined, upload: undefined, response: undefined, end: undefined, error: 1761847230745, abort: undefined, phases: { wait: 4, dns: 1, tcp: 3, tls: undefined, request: undefined, firstByte: undefined, download: undefined, total: 15 } }, options: { request: undefined, agent: { http: undefined, https: undefined, http2: undefined }, h2session: undefined, decompress: true, timeout: { connect: undefined, lookup: undefined, read: undefined, request: undefined, response: undefined, secureConnect: undefined, send: undefined, socket: undefined }, prefixUrl: '', body: undefined, form: undefined, json: undefined, cookieJar: undefined, ignoreInvalidCookies: [secure], searchParams: undefined, dnsLookup: undefined, dnsCache: undefined, context: {}, hooks: { init: [], beforeRequest: [], beforeError: [], beforeRedirect: [], beforeRetry: [], beforeCache: [], afterResponse: [] }, followRedirect: true, maxRedirects: 10, cache: undefined, throwHttpErrors: true, username: '', password: '', http2: [secure], allowGetBody: [secure], copyPipedHeaders: true, headers: { 'user-agent': 'got (https://github.com/sindresorhus/got)', 'private-token': '[secure]', accept: 'application/json', 'accept-encoding': 'gzip, deflate, br, zstd' }, methodRewriting: [secure], dnsLookupIpVersion: undefined, parseJson: [Function: parse], stringifyJson: [Function: stringify], retry: { limit: 2, methods: [ 'GET', 'PUT', 'HEAD', 'DELETE', 'OPTIONS', 'TRACE' ], statusCodes: [ 408, 413, 429, 500, 502, 503, 504, 521, 522, 524 ], errorCodes: [ 'ETIMEDOUT', 'ECONNRESET', 'EADDRINUSE', 'ECONNREFUSED', 'EPIPE', 'ENOTFOUND', 'ENETUNREACH', 'EAI_AGAIN' ], maxRetryAfter: undefined, calculateDelay: [Function: calculateDelay], backoffLimit: Infinity, noise: 100, enforceRetryRules: [secure] }, localAddress: undefined, method: 'GET', createConnection: undefined, cacheOptions: { shared: undefined, cacheHeuristic: undefined, immutableMinTimeToLive: undefined, ignoreCargoCult: undefined }, https: { alpnProtocols: undefined, rejectUnauthorized: undefined, checkServerIdentity: undefined, serverName: undefined, certificateAuthority: undefined, key: undefined, certificate: undefined, passphrase: undefined, pfx: undefined, ciphers: undefined, honorCipherOrder: undefined, minVersion: undefined, maxVersion: undefined, signatureAlgorithms: undefined, tlsSessionLifetime: undefined, dhparam: undefined, ecdhCurve: undefined, certificateRevocationLists: undefined, secureOptions: undefined }, encoding: undefined, resolveBodyOnly: [secure], isStream: [secure], responseType: 'text', url: URL { href: 'https://gitlab.example.com/api/v4/projects/1', origin: 'https://gitlab.example.com', protocol: 'https:', username: '', password: '', host: 'gitlab.example.com', hostname: 'gitlab.example.com', port: '', pathname: '/api/v4/projects/1', search: '', searchParams: URLSearchParams {}, hash: '' }, pagination: { transform: [Function: transform], paginate: [Function: paginate], filter: [Function: filter], shouldContinue: [Function: shouldContinue], countLimit: Infinity, backoff: 0, requestLimit: 10000, stackAllItems: [secure] }, setHost: true, maxHeaderSize: undefined, signal: undefined, enableUnixSockets: [secure], strictContentLength: [secure] }, pluginName: '@semantic-release/gitlab', [cause]: Error: self-signed certificate; if the root CA is installed locally, try running Node.js with --use-system-ca at TLSSocket.onConnectSecure (node:_tls_wrap:1631:34) at TLSSocket.emit (node:events:508:28) at TLSSocket._finishInit (node:_tls_wrap:1077:8) at ssl.onhandshakedone (node:_tls_wrap:863:12) { code: 'DEPTH_ZERO_SELF_SIGNED_CERT' } } AggregateError: RequestError: self-signed certificate; if the root CA is installed locally, try running Node.js with --use-system-ca at ClientRequest.<anonymous> (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/got/dist/source/core/index.js:901:107) at file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/lib/plugins/pipeline.js:55:13 at process.processTicksAndRejections (node:internal/process/task_queues:105:5) at async pluginsConfigAccumulator.<computed> [as verifyConditions] (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/lib/plugins/index.js:87:11) at async run (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/index.js:106:3) at async Module.default (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/index.js:278:22) at async default (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/semantic-release/cli.js:55:5) { errors: [ RequestError: self-signed certificate; if the root CA is installed locally, try running Node.js with --use-system-ca at ClientRequest.<anonymous> (file:///mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/node_modules/got/dist/source/core/index.js:901:107) at Object.onceWrapper (node:events:623:26) at ClientRequest.emit (node:events:520:35) at emitErrorEvent (node:_http_client:107:11) at TLSSocket.socketErrorListener (node:_http_client:574:5) at TLSSocket.emit (node:events:508:28) at emitErrorNT (node:internal/streams/destroy:170:8) at emitErrorCloseNT (node:internal/streams/destroy:129:3) at process.processTicksAndRejections (node:internal/process/task_queues:90:21) at TLSSocket.onConnectSecure (node:_tls_wrap:1631:34) ... 2 lines matching cause stack trace ... at ssl.onhandshakedone (node:_tls_wrap:863:12) { code: 'DEPTH_ZERO_SELF_SIGNED_CERT', input: undefined, timings: [Object], options: { request: undefined, agent: { http: undefined, https: undefined, http2: undefined }, h2session: undefined, decompress: true, timeout: { connect: undefined, lookup: undefined, read: undefined, request: undefined, response: undefined, secureConnect: undefined, send: undefined, socket: undefined }, prefixUrl: '', body: undefined, form: undefined, json: undefined, cookieJar: undefined, ignoreInvalidCookies: [secure], searchParams: undefined, dnsLookup: undefined, dnsCache: undefined, context: {}, hooks: { init: [], beforeRequest: [], beforeError: [], beforeRedirect: [], beforeRetry: [], beforeCache: [], afterResponse: [] }, followRedirect: true, maxRedirects: 10, cache: undefined, throwHttpErrors: true, username: '', password: '', http2: [secure], allowGetBody: [secure], copyPipedHeaders: true, headers: { 'user-agent': 'got (https://github.com/sindresorhus/got)', 'private-token': '[secure]', accept: 'application/json', 'accept-encoding': 'gzip, deflate, br, zstd' }, methodRewriting: [secure], dnsLookupIpVersion: undefined, parseJson: [Function: parse], stringifyJson: [Function: stringify], retry: { limit: 2, methods: [ 'GET', 'PUT', 'HEAD', 'DELETE', 'OPTIONS', 'TRACE' ], statusCodes: [ 408, 413, 429, 500, 502, 503, 504, 521, 522, 524 ], errorCodes: [ 'ETIMEDOUT', 'ECONNRESET', 'EADDRINUSE', 'ECONNREFUSED', 'EPIPE', 'ENOTFOUND', 'ENETUNREACH', 'EAI_AGAIN' ], maxRetryAfter: undefined, calculateDelay: [Function: calculateDelay], backoffLimit: Infinity, noise: 100, enforceRetryRules: [secure] }, localAddress: undefined, method: 'GET', createConnection: undefined, cacheOptions: { shared: undefined, cacheHeuristic: undefined, immutableMinTimeToLive: undefined, ignoreCargoCult: undefined }, https: { alpnProtocols: undefined, rejectUnauthorized: undefined, checkServerIdentity: undefined, serverName: undefined, certificateAuthority: undefined, key: undefined, certificate: undefined, passphrase: undefined, pfx: undefined, ciphers: undefined, honorCipherOrder: undefined, minVersion: undefined, maxVersion: undefined, signatureAlgorithms: undefined, tlsSessionLifetime: undefined, dhparam: undefined, ecdhCurve: undefined, certificateRevocationLists: undefined, secureOptions: undefined }, encoding: undefined, resolveBodyOnly: [secure], isStream: [secure], responseType: 'text', url: URL { href: 'https://gitlab.example.com/api/v4/projects/1', origin: 'https://gitlab.example.com', protocol: 'https:', username: '', password: '', host: 'gitlab.example.com', hostname: 'gitlab.example.com', port: '', pathname: '/api/v4/projects/1', search: '', searchParams: URLSearchParams {}, hash: '' }, pagination: { transform: [Function: transform], paginate: [Function: paginate], filter: [Function: filter], shouldContinue: [Function: shouldContinue], countLimit: Infinity, backoff: 0, requestLimit: 10000, stackAllItems: [secure] }, setHost: true, maxHeaderSize: undefined, signal: undefined, enableUnixSockets: [secure], strictContentLength: [secure] }, pluginName: '@semantic-release/gitlab', [cause]: [Error] } ] } Cleaning up project directory and file based variables 00:01 ERROR: Job failed: exit status 1
```
#### ***in summary***:
```bash
[9:30:30 PM] [semantic-release] › ✘ An error occurred while running semantic-release: RequestError: self-signed certificate;
```

### ***~Fix~***

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

stages:
  - build
  - push
  - release
#1
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
    - VERSION=$(git describe --tags --abbrev=0 || echo "latest")
    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$VERSION
    - docker push $IMAGE_NAME:$TAG
    - docker push $IMAGE_NAME:$VERSION
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA

    - docker pull $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker pull $IMAGE_NAME:$VERSION

  only:
    - develop
#3 Updated
release:
  stage: release
  image: node:24-bullseye
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
#4 (Also add the cleanup stage for semantic versioning to delete the older versions and keeping the last 5 new versions)
cleanup_old_releases:
  stage: cleanup
  image: ubuntu:22.04
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

### 1 - Commit and Push (fix:)
##### (A) Enter this in the Commit message:
```bash
fix: changing patch (for changing the patch version)
```

##### (B) Then:
```
Click the dropdown (˅) -> Click the "Create new branch and commit" -> Write "develop -> Enter
```
### ***Pipelines:***
### (A) Build Stage
- **Status**: Passed
- **Executor**: Shell

```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:03
Gitaly correlation ID: 01K95P3KWNB4BNSWYYHQGN3SMY
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 6892fc04 as detached HEAD (ref is develop)...
Skipping Git submodules setup
Executing "step_script" stage of the job script 00:45
$ docker build -t $IMAGE_NAME:$TAG .
#0 building with "default" instance using docker driver
#1 [internal] load build definition from Dockerfile
#1 transferring dockerfile:
#1 transferring dockerfile: 275B 0.1s done
#1 DONE 0.9s
#2 [internal] load metadata for docker.io/library/golang:1.22
#2 DONE 0.0s
#3 [internal] load metadata for docker.io/library/alpine:latest
#3 DONE 0.0s
#4 [internal] load .dockerignore
#4 transferring context:
#4 transferring context: 2B 0.0s done
#4 DONE 1.1s
#5 [builder 1/5] FROM docker.io/library/golang:1.22
#5 DONE 0.0s
#6 [internal] load build context
#6 DONE 0.0s
#7 [stage-1 1/3] FROM docker.io/library/alpine:latest
#7 DONE 0.0s
#6 [internal] load build context
#6 transferring context: 59.39kB 3.6s
#6 transferring context: 420.09kB 4.7s done
#6 DONE 5.3s
#8 [builder 2/5] WORKDIR /src
#8 CACHED
#9 [builder 3/5] COPY . .
#9 DONE 2.8s
#10 [builder 4/5] RUN go mod init example.com/hello || true
#10 5.271 go: creating new go.mod: module example.com/hello
#10 5.320 go: to add module requirements and sums:
#10 5.320 	go mod tidy
#10 DONE 5.8s
#11 [builder 5/5] RUN go build -o app .
#11 DONE 15.9s
#12 [stage-1 2/3] WORKDIR /root/
#12 CACHED
#13 [stage-1 3/3] COPY --from=builder /src/app .
#13 DONE 1.5s
#14 exporting to image
#14 exporting layers
#14 exporting layers 1.0s done
#14 writing image sha256:84ddfc9b8281902c783a0f734db130ac521e8d2c5d7a735f059961b89855437b
#14 writing image sha256:84ddfc9b8281902c783a0f734db130ac521e8d2c5d7a735f059961b89855437b 0.2s done
#14 naming to registry.gitlab.example.com:5050/mygroup/hello-go:latest
#14 naming to registry.gitlab.example.com:5050/mygroup/hello-go:latest 0.3s done
#14 DONE 2.0s
$ docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
Cleaning up project directory and file based variables 00:00
Job succeeded
```

### (B) Push Stage
- **Status**: Passed

```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:02
Gitaly correlation ID: 01K95P55PD86A0ZCJCP3KPW4Q2
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 6892fc04 as detached HEAD (ref is develop)...
Skipping Git submodules setup
Executing "step_script" stage of the job script 00:06
$ echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" registry.gitlab.example.com:5050 --password-stdin
Login Succeeded
$ VERSION=$(git describe --tags --abbrev=0 || echo "latest")
$ docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$VERSION
$ docker push $IMAGE_NAME:$TAG
The push refers to repository [registry.gitlab.example.com:5050/mygroup/hello-go]
46311e6d3481: Preparing
5f70bf18a086: Preparing
256f393e029f: Preparing
256f393e029f: Layer already exists
5f70bf18a086: Layer already exists
46311e6d3481: Pushed
latest: digest: sha256:40eeb0b47360d7e26d0ffc73786dad706e7a72b33fc6ea6c5c1be8d866ea9a5a size: 945
$ docker push $IMAGE_NAME:$VERSION
The push refers to repository [registry.gitlab.example.com:5050/mygroup/hello-go]
46311e6d3481: Preparing
5f70bf18a086: Preparing
256f393e029f: Preparing
46311e6d3481: Layer already exists
256f393e029f: Layer already exists
5f70bf18a086: Layer already exists
v1.3.0: digest: sha256:40eeb0b47360d7e26d0ffc73786dad706e7a72b33fc6ea6c5c1be8d866ea9a5a size: 945
$ docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
The push refers to repository [registry.gitlab.example.com:5050/mygroup/hello-go]
46311e6d3481: Preparing
5f70bf18a086: Preparing
256f393e029f: Preparing
46311e6d3481: Layer already exists
256f393e029f: Layer already exists
5f70bf18a086: Layer already exists
6892fc04: digest: sha256:40eeb0b47360d7e26d0ffc73786dad706e7a72b33fc6ea6c5c1be8d866ea9a5a size: 945
$ docker pull $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
6892fc04: Pulling from mygroup/hello-go
Digest: sha256:40eeb0b47360d7e26d0ffc73786dad706e7a72b33fc6ea6c5c1be8d866ea9a5a
Status: Image is up to date for registry.gitlab.example.com:5050/mygroup/hello-go:6892fc04
registry.gitlab.example.com:5050/mygroup/hello-go:6892fc04
$ docker pull $IMAGE_NAME:$VERSION
v1.3.0: Pulling from mygroup/hello-go
Digest: sha256:40eeb0b47360d7e26d0ffc73786dad706e7a72b33fc6ea6c5c1be8d866ea9a5a
Status: Image is up to date for registry.gitlab.example.com:5050/mygroup/hello-go:v1.3.0
registry.gitlab.example.com:5050/mygroup/hello-go:v1.3.0
Cleaning up project directory and file based variables 00:00
Job succeeded
```

### (C) Release Stage
- **Status**: Passed

```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:04
Gitaly correlation ID: 01K95P5GTYKYMPHBQHWXN8NSBT
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 6892fc04 as detached HEAD (ref is develop)...
Skipping Git submodules setup
Executing "step_script" stage of the job script 02:34
$ npm ci
npm warn deprecated semver-diff@5.0.0: Deprecated as the semver package now supports this built-in.
added 371 packages, and audited 572 packages in 1m
115 packages are looking for funding
  run `npm fund` for details
4 moderate severity vulnerabilities
To address all issues (including breaking changes), run:
  npm audit fix --force
Run `npm audit` for details.
$ npm install --save-dev conventional-changelog-conventionalcommits
up to date, audited 572 packages in 8s
115 packages are looking for funding
  run `npm fund` for details
4 moderate severity vulnerabilities
To address all issues (including breaking changes), run:
  npm audit fix --force
Run `npm audit` for details.
$ npx semantic-release
[11:54:29 PM] [semantic-release] › ℹ  Running semantic-release version 25.0.1
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "verifyConditions" from "@semantic-release/gitlab"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "verifyConditions" from "@semantic-release/changelog"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "verifyConditions" from "@semantic-release/git"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "analyzeCommits" from "@semantic-release/commit-analyzer"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "generateNotes" from "@semantic-release/release-notes-generator"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "prepare" from "@semantic-release/changelog"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "prepare" from "@semantic-release/git"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "publish" from "@semantic-release/gitlab"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "success" from "@semantic-release/gitlab"
[11:54:44 PM] [semantic-release] › ✔  Loaded plugin "fail" from "@semantic-release/gitlab"
[11:54:53 PM] [semantic-release] › ✔  Run automated release from branch develop on repository https://gitlab.example.com/mygroup/hello-go.git
[11:54:53 PM] [semantic-release] › ✔  Allowed to push to the Git repository
[11:54:53 PM] [semantic-release] › ℹ  Start step "verifyConditions" of plugin "@semantic-release/gitlab"
[11:54:54 PM] [semantic-release] [@semantic-release/gitlab] › ℹ  Verify GitLab authentication (https://gitlab.example.com/api/v4)
[11:54:54 PM] [semantic-release] › ✔  Completed step "verifyConditions" of plugin "@semantic-release/gitlab"
[11:54:54 PM] [semantic-release] › ℹ  Start step "verifyConditions" of plugin "@semantic-release/changelog"
[11:54:54 PM] [semantic-release] › ✔  Completed step "verifyConditions" of plugin "@semantic-release/changelog"
[11:54:54 PM] [semantic-release] › ℹ  Start step "verifyConditions" of plugin "@semantic-release/git"
[11:54:54 PM] [semantic-release] › ✔  Completed step "verifyConditions" of plugin "@semantic-release/git"
[11:54:54 PM] [semantic-release] › ℹ  Found git tag v1.3.0 associated with version 1.3.0 on branch develop
[11:54:54 PM] [semantic-release] › ℹ  Found 4 commits since last release
[11:54:54 PM] [semantic-release] › ℹ  Start step "analyzeCommits" of plugin "@semantic-release/commit-analyzer"
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  Analyzing commit: fix: changing patch
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  The release type for the commit is patch
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  Analyzing commit: Update file .gitlab-ci.yml
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  The commit should not trigger a release
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  Analyzing commit: Update file .gitlab-ci.yml
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  The commit should not trigger a release
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  Analyzing commit: Update file .releaserc.json
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  The commit should not trigger a release
[11:54:55 PM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  Analysis of 4 commits complete: patch release
[11:54:55 PM] [semantic-release] › ✔  Completed step "analyzeCommits" of plugin "@semantic-release/commit-analyzer"
[11:54:55 PM] [semantic-release] › ℹ  The next release version is 1.3.1
[11:54:55 PM] [semantic-release] › ℹ  Start step "generateNotes" of plugin "@semantic-release/release-notes-generator"
[11:54:55 PM] [semantic-release] › ✔  Completed step "generateNotes" of plugin "@semantic-release/release-notes-generator"
[11:54:55 PM] [semantic-release] › ℹ  Start step "prepare" of plugin "@semantic-release/changelog"
[11:54:55 PM] [semantic-release] [@semantic-release/changelog] › ℹ  Update /mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/CHANGELOG.md
[11:54:55 PM] [semantic-release] › ✔  Completed step "prepare" of plugin "@semantic-release/changelog"
[11:54:55 PM] [semantic-release] › ℹ  Start step "prepare" of plugin "@semantic-release/git"
[11:55:02 PM] [semantic-release] [@semantic-release/git] › ℹ  Found 1 file(s) to commit
[11:55:07 PM] [semantic-release] [@semantic-release/git] › ℹ  Prepared Git release: v1.3.1
[11:55:07 PM] [semantic-release] › ✔  Completed step "prepare" of plugin "@semantic-release/git"
[11:55:07 PM] [semantic-release] › ℹ  Start step "generateNotes" of plugin "@semantic-release/release-notes-generator"
[11:55:07 PM] [semantic-release] › ✔  Completed step "generateNotes" of plugin "@semantic-release/release-notes-generator"
[11:55:12 PM] [semantic-release] › ✔  Created tag v1.3.1
[11:55:12 PM] [semantic-release] › ℹ  Start step "publish" of plugin "@semantic-release/gitlab"
[11:55:14 PM] [semantic-release] [@semantic-release/gitlab] › ℹ  Published GitLab release: v1.3.1
[11:55:14 PM] [semantic-release] › ✔  Completed step "publish" of plugin "@semantic-release/gitlab"
[11:55:14 PM] [semantic-release] › ℹ  Start step "success" of plugin "@semantic-release/gitlab"
[11:55:14 PM] [semantic-release] › ✔  Completed step "success" of plugin "@semantic-release/gitlab"
[11:55:14 PM] [semantic-release] › ✔  Published release 1.3.1 on develop channel
Cleaning up project directory and file based variables 00:00
Job succeeded
```

### (D) Cleanup Stage
- **Status**: Passed

```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:41
Gitaly correlation ID: 01K95PAF6XDXV5MJA32XZ9M3JT
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 6892fc04 as detached HEAD (ref is develop)...
Removing node_modules/
Skipping Git submodules setup
Executing "step_script" stage of the job script 00:07
$ if ! command -v git &> /dev/null; then echo "Installing git..."; apt-get update && apt-get install -y git; fi
$ git config --global user.name "GitLab CI"
$ git config --global user.email "ci-runner@gitlab.example.com"
$ git remote set-url origin https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/mygroup/hello-go.git
$ echo "Fetching all tags..."
Fetching all tags...
$ git fetch --tags --force
$ echo "All tags (newest first):"
All tags (newest first):
$ git tag --sort=-creatordate
v1.3.1
v1.3.0
v1.2.3
v1.2.2
v1.2.1
v1.2.0
$ TAGS=$(git tag --sort=-creatordate | tail -n +6)
$ if [ -n "$TAGS" ]; then # collapsed multi-line command
Deleting old tags (remote + local):
Deleting tag: v1.2.0 (remote + local)
To https://gitlab.example.com/mygroup/hello-go.git
 - [deleted]         v1.2.0
Deleted tag 'v1.2.0' (was 6515cdb)
$ echo "Fetching tags again after cleanup..."
Fetching tags again after cleanup...
$ git fetch --tags --force
$ echo "Remaining tags (newest first):"
Remaining tags (newest first):
$ git tag --sort=-creatordate
v1.3.1
v1.3.0
v1.2.3
v1.2.2
v1.2.1
Cleaning up project directory and file based variables 00:00
Job succeeded
```

### 2 - Commit and Push (feat:)
##### Enter this in the Commit message:
```bash
feat: changing minor (for changing the minor version)
```

### ***Pipelines:***
### (A) Build Stage
- **Status**: Passed
- **Executor**: Shell

```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:03
Gitaly correlation ID: 01K95PHW7KP64B2JQ8HB8F9KQW
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 66bf0117 as detached HEAD (ref is develop)...
Skipping Git submodules setup
Executing "step_script" stage of the job script 00:38
$ docker build -t $IMAGE_NAME:$TAG .
#0 building with "default" instance using docker driver
#1 [internal] load build definition from Dockerfile
#1 transferring dockerfile:
#1 transferring dockerfile: 275B 0.1s done
#1 DONE 0.5s
#2 [internal] load metadata for docker.io/library/golang:1.22
#2 DONE 0.0s
#3 [internal] load metadata for docker.io/library/alpine:latest
#3 DONE 0.0s
#4 [internal] load .dockerignore
#4 transferring context:
#4 transferring context: 2B 0.1s done
#4 DONE 0.7s
#5 [builder 1/5] FROM docker.io/library/golang:1.22
#5 DONE 0.0s
#6 [stage-1 1/3] FROM docker.io/library/alpine:latest
#6 DONE 0.0s
#7 [internal] load build context
#7 DONE 0.0s
#7 [internal] load build context
#7 transferring context: 62.63kB 3.8s
#7 transferring context: 429.22kB 4.9s done
#7 DONE 5.2s
#8 [builder 2/5] WORKDIR /src
#8 CACHED
#9 [builder 3/5] COPY . .
#9 DONE 3.4s
#10 [builder 4/5] RUN go mod init example.com/hello || true
#10 3.645 go: creating new go.mod: module example.com/hello
#10 3.697 go: to add module requirements and sums:
#10 3.697 	go mod tidy
#10 DONE 4.3s
#11 [builder 5/5] RUN go build -o app .
#11 DONE 14.8s
#12 [stage-1 2/3] WORKDIR /root/
#12 CACHED
#13 [stage-1 3/3] COPY --from=builder /src/app .
#13 DONE 1.6s
#14 exporting to image
#14 exporting layers
#14 exporting layers 0.8s done
#14 writing image sha256:2cb3dc256bf6f562785d1c612a60ed10011f347f55eb2e0331321da41939a8ae
#14 writing image sha256:2cb3dc256bf6f562785d1c612a60ed10011f347f55eb2e0331321da41939a8ae 0.1s done
#14 naming to registry.gitlab.example.com:5050/mygroup/hello-go:latest 0.1s done
#14 DONE 1.2s
$ docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
Cleaning up project directory and file based variables 00:01
Job succeeded
```

### (B) Push Stage
- **Status**: Passed

```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:02
Gitaly correlation ID: 01K95PP5BE64YK14VFTMD9B8D7
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 66bf0117 as detached HEAD (ref is develop)...
Skipping Git submodules setup
Executing "step_script" stage of the job script 00:05
$ echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" registry.gitlab.example.com:5050 --password-stdin
Login Succeeded
$ VERSION=$(git describe --tags --abbrev=0 || echo "latest")
$ docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$VERSION
$ docker push $IMAGE_NAME:$TAG
The push refers to repository [registry.gitlab.example.com:5050/mygroup/hello-go]
3e9d50695e85: Preparing
5f70bf18a086: Preparing
256f393e029f: Preparing
5f70bf18a086: Layer already exists
256f393e029f: Layer already exists
3e9d50695e85: Layer already exists
latest: digest: sha256:22761e2fe1baf8a358160e79cd935cee23912db4ab0b4443e24415421ceb1e31 size: 945
$ docker push $IMAGE_NAME:$VERSION
The push refers to repository [registry.gitlab.example.com:5050/mygroup/hello-go]
3e9d50695e85: Preparing
5f70bf18a086: Preparing
256f393e029f: Preparing
5f70bf18a086: Layer already exists
256f393e029f: Layer already exists
3e9d50695e85: Layer already exists
v1.3.1: digest: sha256:22761e2fe1baf8a358160e79cd935cee23912db4ab0b4443e24415421ceb1e31 size: 945
$ docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
The push refers to repository [registry.gitlab.example.com:5050/mygroup/hello-go]
3e9d50695e85: Preparing
5f70bf18a086: Preparing
256f393e029f: Preparing
256f393e029f: Layer already exists
3e9d50695e85: Layer already exists
5f70bf18a086: Layer already exists
66bf0117: digest: sha256:22761e2fe1baf8a358160e79cd935cee23912db4ab0b4443e24415421ceb1e31 size: 945
$ docker pull $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
66bf0117: Pulling from mygroup/hello-go
Digest: sha256:22761e2fe1baf8a358160e79cd935cee23912db4ab0b4443e24415421ceb1e31
Status: Image is up to date for registry.gitlab.example.com:5050/mygroup/hello-go:66bf0117
registry.gitlab.example.com:5050/mygroup/hello-go:66bf0117
$ docker pull $IMAGE_NAME:$VERSION
v1.3.1: Pulling from mygroup/hello-go
Digest: sha256:22761e2fe1baf8a358160e79cd935cee23912db4ab0b4443e24415421ceb1e31
Status: Image is up to date for registry.gitlab.example.com:5050/mygroup/hello-go:v1.3.1
registry.gitlab.example.com:5050/mygroup/hello-go:v1.3.1
Cleaning up project directory and file based variables 00:00
Job succeeded
```

### (C) Release Stage
- **Status**: Passed

```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:03
Gitaly correlation ID: 01K95PPFER01XSXNRZ4207XHP6
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 66bf0117 as detached HEAD (ref is develop)...
Skipping Git submodules setup
Executing "step_script" stage of the job script 02:35
$ npm ci
npm warn deprecated semver-diff@5.0.0: Deprecated as the semver package now supports this built-in.
added 371 packages, and audited 572 packages in 1m
115 packages are looking for funding
  run `npm fund` for details
4 moderate severity vulnerabilities
To address all issues (including breaking changes), run:
  npm audit fix --force
Run `npm audit` for details.
$ npm install --save-dev conventional-changelog-conventionalcommits
up to date, audited 572 packages in 8s
115 packages are looking for funding
  run `npm fund` for details
4 moderate severity vulnerabilities
To address all issues (including breaking changes), run:
  npm audit fix --force
Run `npm audit` for details.
$ npx semantic-release
[12:03:44 AM] [semantic-release] › ℹ  Running semantic-release version 25.0.1
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "verifyConditions" from "@semantic-release/gitlab"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "verifyConditions" from "@semantic-release/changelog"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "verifyConditions" from "@semantic-release/git"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "analyzeCommits" from "@semantic-release/commit-analyzer"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "generateNotes" from "@semantic-release/release-notes-generator"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "prepare" from "@semantic-release/changelog"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "prepare" from "@semantic-release/git"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "publish" from "@semantic-release/gitlab"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "success" from "@semantic-release/gitlab"
[12:04:00 AM] [semantic-release] › ✔  Loaded plugin "fail" from "@semantic-release/gitlab"
[12:04:11 AM] [semantic-release] › ✔  Run automated release from branch develop on repository https://gitlab.example.com/mygroup/hello-go.git
[12:04:12 AM] [semantic-release] › ✔  Allowed to push to the Git repository
[12:04:12 AM] [semantic-release] › ℹ  Start step "verifyConditions" of plugin "@semantic-release/gitlab"
[12:04:12 AM] [semantic-release] [@semantic-release/gitlab] › ℹ  Verify GitLab authentication (https://gitlab.example.com/api/v4)
[12:04:12 AM] [semantic-release] › ✔  Completed step "verifyConditions" of plugin "@semantic-release/gitlab"
[12:04:12 AM] [semantic-release] › ℹ  Start step "verifyConditions" of plugin "@semantic-release/changelog"
[12:04:12 AM] [semantic-release] › ✔  Completed step "verifyConditions" of plugin "@semantic-release/changelog"
[12:04:12 AM] [semantic-release] › ℹ  Start step "verifyConditions" of plugin "@semantic-release/git"
[12:04:12 AM] [semantic-release] › ✔  Completed step "verifyConditions" of plugin "@semantic-release/git"
[12:04:12 AM] [semantic-release] › ℹ  Found git tag v1.3.1 associated with version 1.3.1 on branch develop
[12:04:13 AM] [semantic-release] › ℹ  Found 1 commits since last release
[12:04:13 AM] [semantic-release] › ℹ  Start step "analyzeCommits" of plugin "@semantic-release/commit-analyzer"
[12:04:13 AM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  Analyzing commit: feat: changing minor
[12:04:13 AM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  The release type for the commit is minor
[12:04:13 AM] [semantic-release] [@semantic-release/commit-analyzer] › ℹ  Analysis of 1 commits complete: minor release
[12:04:13 AM] [semantic-release] › ✔  Completed step "analyzeCommits" of plugin "@semantic-release/commit-analyzer"
[12:04:13 AM] [semantic-release] › ℹ  The next release version is 1.4.0
[12:04:13 AM] [semantic-release] › ℹ  Start step "generateNotes" of plugin "@semantic-release/release-notes-generator"
[12:04:13 AM] [semantic-release] › ✔  Completed step "generateNotes" of plugin "@semantic-release/release-notes-generator"
[12:04:13 AM] [semantic-release] › ℹ  Start step "prepare" of plugin "@semantic-release/changelog"
[12:04:13 AM] [semantic-release] [@semantic-release/changelog] › ℹ  Update /mnt/c/Users/Webhouse/builds/[secure]/0/mygroup/hello-go/CHANGELOG.md
[12:04:13 AM] [semantic-release] › ✔  Completed step "prepare" of plugin "@semantic-release/changelog"
[12:04:13 AM] [semantic-release] › ℹ  Start step "prepare" of plugin "@semantic-release/git"
[12:04:20 AM] [semantic-release] [@semantic-release/git] › ℹ  Found 1 file(s) to commit
[12:04:23 AM] [semantic-release] [@semantic-release/git] › ℹ  Prepared Git release: v1.4.0
[12:04:23 AM] [semantic-release] › ✔  Completed step "prepare" of plugin "@semantic-release/git"
[12:04:23 AM] [semantic-release] › ℹ  Start step "generateNotes" of plugin "@semantic-release/release-notes-generator"
[12:04:24 AM] [semantic-release] › ✔  Completed step "generateNotes" of plugin "@semantic-release/release-notes-generator"
[12:04:28 AM] [semantic-release] › ✔  Created tag v1.4.0
[12:04:28 AM] [semantic-release] › ℹ  Start step "publish" of plugin "@semantic-release/gitlab"
[12:04:29 AM] [semantic-release] [@semantic-release/gitlab] › ℹ  Published GitLab release: v1.4.0
[12:04:29 AM] [semantic-release] › ✔  Completed step "publish" of plugin "@semantic-release/gitlab"
[12:04:29 AM] [semantic-release] › ℹ  Start step "success" of plugin "@semantic-release/gitlab"
[12:04:29 AM] [semantic-release] › ✔  Completed step "success" of plugin "@semantic-release/gitlab"
[12:04:29 AM] [semantic-release] › ✔  Published release 1.4.0 on develop channel
Cleaning up project directory and file based variables 00:00
Job succeeded
```

### (D) Cleanup Stage
- **Status**: Passed

```bash
Running with gitlab-runner 18.5.0 (bda84871)
  on shell-runner 7vEJkJxfR, system ID: s_d3e36046c4ec
Preparing the "shell" executor 00:00
Using Shell (bash) executor...
Preparing environment 00:00
Running on Aceraspire7-Kite...
Getting source from Git repository 00:42
Gitaly correlation ID: 01K95PVD1VHYNPNFBMF9S2YN57
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /mnt/c/Users/Webhouse/builds/7vEJkJxfR/0/mygroup/hello-go/.git/
Checking out 66bf0117 as detached HEAD (ref is develop)...
Removing node_modules/
Skipping Git submodules setup
Executing "step_script" stage of the job script 00:06
$ if ! command -v git &> /dev/null; then echo "Installing git..."; apt-get update && apt-get install -y git; fi
$ git config --global user.name "GitLab CI"
$ git config --global user.email "ci-runner@gitlab.example.com"
$ git remote set-url origin https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/mygroup/hello-go.git
$ echo "Fetching all tags..."
Fetching all tags...
$ git fetch --tags --force
$ echo "All tags (newest first):"
All tags (newest first):
$ git tag --sort=-creatordate
v1.4.0
v1.3.1
v1.3.0
v1.2.3
v1.2.2
v1.2.1
$ TAGS=$(git tag --sort=-creatordate | tail -n +6)
$ if [ -n "$TAGS" ]; then # collapsed multi-line command
Deleting old tags (remote + local):
Deleting tag: v1.2.1 (remote + local)
To https://gitlab.example.com/mygroup/hello-go.git
 - [deleted]         v1.2.1
Deleted tag 'v1.2.1' (was 0fc19d5)
$ echo "Fetching tags again after cleanup..."
Fetching tags again after cleanup...
$ git fetch --tags --force
$ echo "Remaining tags (newest first):"
Remaining tags (newest first):
$ git tag --sort=-creatordate
v1.4.0
v1.3.1
v1.3.0
v1.2.3
v1.2.2
Cleaning up project directory and file based variables 00:00
Job succeeded
```
