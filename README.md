# sysclig-cli-scanner-pre-commit-hook

## Overview

This is a pre-commit hook that will scan will build an image based on the local Dockerfile and scan it with Sysdig Secure. If the scan fails, the commit will be aborted.

>NOTE: This is a proof of concept. It's not meant to be used in production, rather as an example of what can be done with Sysdig Secure and Git hooks.


## Setup a Sysdig Secure Trial Account

Easily done at [https://sysdig.com/start-free/](https://sysdig.com/start-free/).

### Create a Pipeline Policy

Create a pipeline policy that will be used to evaluate the results of the scan. Here's an example of a policy that will fail if there are any high or greater vulnerabilities that are fixable.

![pipeline policy](/img/pipeline-policy.jpg)


## Configure Git Hooks

Make the git hooks directory.

```
mkdir .githooks
```

Download the pre-commit hook.

```
curl --silent -o .githooks/pre-commit https://raw.githubusercontent.com/ccollicutt/sysdig-cli-scanner-pre-commit-hook/main/.githooks/pre-commit
chmod +x .githooks/pre-commit
```

Tell git to use the hooks in this repo.

```
git config core.hooksPath .githooks
```

## Setup Environment Variables

Setup some environment variables. I'm using direnv, but however you want to do it.

```
$ cat .envrc 
export SECURE_API_TOKEN=<REDACTED>
export IMAGE_NAME=sysdig-cli-scanner-image
export SYSDIG_API_URL=https://app.us4.sysdig.com
export SYSDIG_POLICY_NAME=severity-greater-than-high-and-fixable
```

## Add a Dockerfile and Commit

If no Dockerfile has been added, nothing happens.

```
$ git commit -m "some commit message"
No staged changes. Skipping pre-commit hook.
On branch main
Your branch is up to date with 'origin/main'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	README.md

nothing added to commit but untracked files present (use "git add" to track)
```

If we add a Dockerfile, the image will be built and then scanned by Sysdig Secure.

```
$ git add Dockerfile
$ git commit -m "upate Dockerfile"
==================================================
Building the Docker image...
[+] Building 0.0s (6/6) FINISHED                                    
 => [internal] load .dockerignore                              0.0s
 => => transferring context: 2B                                0.0s
 => [internal] load build definition from Dockerfile           0.0s
 => => transferring dockerfile: 81B                            0.0s
 => [internal] load metadata for docker.io/library/python:2.7  0.0s
 => [1/2] FROM docker.io/library/python:2.7                    0.0s
 => CACHED [2/2] RUN echo hi > /root/hi.txt2                   0.0s
 => exporting to image                                         0.0s
 => => exporting layers                                        0.0s
 => => writing image sha256:b8f6630458f7874de3829321636748829  0.0s
 => => naming to docker.io/library/sysdig-cli-scanner-image:1  0.0s
Docker build successful.
==================================================
Running Sysdig Secure scan...
2023-05-09T06:45:03-04:00 Starting analysis with Sysdig scanner version 1.1.1-rc
2023-05-09T06:45:03-04:00 Retrieving vulnerabilities DB...
2023-05-09T06:45:03-04:00 Done, using cached DB
2023-05-09T06:45:03-04:00 Loading vulnerabilities DB...
2023-05-09T06:45:03-04:00 Done
2023-05-09T06:45:04-04:00 Retrieving image...
2023-05-09T06:45:11-04:00 Done
2023-05-09T06:45:11-04:00 Scan started...
2023-05-09T06:45:11-04:00 Uploading result to backend...
2023-05-09T06:45:12-04:00 Done
2023-05-09T06:45:12-04:00 Total execution time 8.328622892s

Type: dockerImage
ImageID: sha256:b8f6630458f7874de3829321636748829b00bab01c58cb9d340380f49303d7d7
Digest: 
BaseOS: debian 10.3
PullString: sysdig-cli-scanner-image:1683629103

3779 vulnerabilities found
176 Critical (169 fixable)
1232 High (1002 fixable)
1388 Medium (1128 fixable)
243 Low (164 fixable)
740 Negligible (0 fixable)

     PACKAGE     TYPE      VERSION       SUGGESTED FIX   CRITICAL  HIGH  MEDIUM  LOW  NEGLIGIBLE  EXPLOIT  
  libwebp-dev     os       0.6.1-2      0.6.1-2+deb10u1     10      1      0      0       1          0     
  libwebp6        os       0.6.1-2      0.6.1-2+deb10u1     10      1      0      0       1          0     
  libwebpdemux2   os       0.6.1-2      0.6.1-2+deb10u1     10      1      0      0       1          0     
  libwebpmux3     os       0.6.1-2      0.6.1-2+deb10u1     10      1      0      0       1          0     
  libexpat1       os   2.2.6-2+deb10u1  2.2.6-2+deb10u2     7       9      1      0       1          0     
  libexpat1-dev   os   2.2.6-2+deb10u1  2.2.6-2+deb10u2     7       9      1      0       1          0     
  libc-bin        os       2.28-10      2.28-10+deb10u2     4       4      4      2       6          0     
  libc-dev-bin    os       2.28-10      2.28-10+deb10u2     4       4      4      2       6          0     
  libc6           os       2.28-10      2.28-10+deb10u2     4       4      4      2       6          0     
  libc6-dev       os       2.28-10      2.28-10+deb10u2     4       4      4      2       6          0     

                            POLICIES EVALUATION
    Policy: severity-greater-than-high-and-fixable FAILED (1171 failures)

Policies evaluation FAILED at 2023-05-09T06:45:12-04:00
Full image results here: https://app.us4.sysdig.com/secure/#/scanning/assets/results/175d73c1f709f3b11f2e4c38b7659860/overview (id 175d73c1f709f3b11f2e4c38b7659860)
Execution logs written to: /home/curtis/working/sysdig-cli-scanner-pre-commit-hook/scan-logs
Sysdig Secure scan failed. Fix the issues before committing.
```

Above this policy has failed. If we fix the issues and commit again, the policy will pass.

If we change the image to be `python:3.11.3-slim` from the old, deprecated, no longer used, great as a insecure demo image `python:2.7`, the policy will pass.

```
$ git commit -m "update"
==================================================
Building the Docker image...
SNIP!
Docker build successful.
==================================================
Running Sysdig Secure scan...
SNIP!
67 vulnerabilities found
1 Critical (0 fixable)
5 High (0 fixable)
4 Medium (0 fixable)
5 Low (0 fixable)
52 Negligible (0 fixable)

                           POLICIES EVALUATION
    Policy: severity-greater-than-high-and-fixable PASSED (0 failures)

Policies evaluation PASSED at 2023-05-09T07:02:58-04:00
SNIP!
```

## IGNORE_DOCKERFILE

If you set `IGNORE_DOCKERFILE` to anything, the pre-commit hook will skip the scan.
