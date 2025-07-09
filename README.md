# Wallarm Solutions Engineer Technical Evaluation

- [Wallarm Solutions Engineer Technical Evaluation](#wallarm-solutions-engineer-technical-evaluation)
  - [📌 Overview](#-overview)
  - [Craig Thomas's Solution](#craig-thomass-solution)
  - [Overall Architecture](#overall-architecture)
  - [🎯 Objectives](#-objectives)
  - [📂 Prerequisites](#-prerequisites)
  - [🚀 Task Breakdown](#-task-breakdown)
    - [1️⃣ Deploy a Wallarm Filtering Node \&\&](#1️⃣-deploy-a-wallarm-filtering-node-)
    - [2️⃣ Set Up a Backend Origin](#2️⃣-set-up-a-backend-origin)
    - [3️⃣ Generate Traffic Using GoTestWAF](#3️⃣-generate-traffic-using-gotestwaf)
    - [Redeploy Wallarm in Blocking Mode](#redeploy-wallarm-in-blocking-mode)
    - [4️⃣ Document Your Process](#4️⃣-document-your-process)
  - [✅ Evaluation Criteria](#-evaluation-criteria)
  - [📬 Submission](#-submission)
  - [ℹ️ Additional Notes](#ℹ️-additional-notes)



## 📌 Overview

Welcome to the **Wallarm Solutions Engineer Technical Evaluation**. This exercise is designed to assess your ability to deploy and configure Wallarm's filtering nodes using a deployment method of your choice, troubleshoot any issues encountered, and document your process effectively. Additionally, we will evaluate your ability to leverage our official documentation to complete the task.

---

## Craig Thomas's Solution
This was the solution, as performed by Craig Thomas. While there are many possible deployments, I have laid out my architecture and overall deployment decisions, steps, and screenshots in the below document. Additionally, as I ran into hurdles or challenges, I documented those.

## Overall Architecture
![Architecture](images/architecture.png)

## 🎯 Objectives

By the end of this evaluation, you should be able to:

✅ Deploy a Wallarm filtering node using a supported method of your choice.  
✅ Configure a backend origin to receive test traffic. (httpbin.org is also acceptable)  
✅ Use the **GoTestWAF** attack simulation tool to generate traffic.  
✅ Document the deployment and troubleshooting process.  
✅ Demonstrate proficiency in using **Wallarm's official documentation**.  

--------

## 📂 Prerequisites
**Desktop Environment**
- I will be using Docker running on my local Mac (Mac silicon, so ARM64 architecture) for the various components of this deployment including (as detailed in the drawing)
**Backend Application/API endpoint**
- I used Postman to connect to and test the HTTPBin container (also tested it against https://httpbin.org)
    - For the local container, I just ran it using `docker run -p 80:80 kennethreitz/httpbin`
    -  I was able to import the collection from Postman for quicker access to the various parts of the API (https://www.postman.com/postman/httpbin/documentation/0bjofuo/httpbin-org-current)
 - I set the `baseURL` variable to `https://httpbin.org` for testing on the web and `http://127.0.0.1` for testing my local API container
    - For testing against `https://httpbin.org`, I successfully got my external IP address
    - For testing against `http://172.0.0.1`, I successfully the IP of my Mac on the Docker Bridge network (in this case `172.17.0.1` or `192.168.65.1`)
    - For testing through the Wallarm proxy, I set `baseURL` to `http:/127.0.0.1:81`, as I was exposing that container on port 81
 - For a simple test, I tested `GET` against `/ip`
 - I also looked at using mockapi.io, but httpbin works well, since the API is fully set up. mockapi.io is nice when building out a quick mock API
 - For this exercise, I did not set up HTTPs
**GoTestWAF**
- I spun this up in a Docker container running in my desktop environment
**Documentation**
- I referenced this for the various deployments and throughout the process
**Wallarm GUI/Account**
- I got the invite from Brandon, and set up my account in the Wallarm tenant: https://us1.my.wallarm.com/ and user account: `cdthomas23@gmail.com`

## 🚀 Task Breakdown
### 1️⃣ Deploy a Wallarm Filtering Node &&
### 2️⃣ Set Up a Backend Origin
- I am using the Docker NGINX-based Image (https://docs.wallarm.com/admin-en/installation-docker-en/)
- Requirements:
 ✅  Docker installed on your host system
 ✅  Access to https://hub.docker.com/r/wallarm/node to download the Docker image. Please ensure the access is not blocked by a firewall
  - Verified I could do a `docker pull wallarm/node`
 ✅  Access to the account with the Administrator role in Wallarm Console in the US Cloud or EU Cloud
  - cdthomas23@gmail.com has the Adminstrator role
  - ![Admin Roles](images/admin.png)
 ✅ Access to https://us1.api.wallarm.com if working with US Wallarm Cloud or to https://api.wallarm.com if working with EU Wallarm Cloud. Please ensure the access is not blocked by a firewall
 ✅ Access to the IP addresses below for downloading updates to attack detection rules and API specifications, as well as retrieving precise IPs for your allowlisted, denylisted, or graylisted countries, regions, or data centers:
 ```
 34.96.64.17
 34.110.183.149
 35.235.66.155
 34.102.90.100
 34.94.156.115
 35.235.115.105
 ```
1. Set up API Token (Settings->API Tokens)
   - Created new token named `DeploymentToken1`
   - Token Usage: `Node Deployment`
   - No expiration
   - Source role: `Deploy`
   - I then pasted this in my Password Manager for later use
2. Start back up the API (running on port 80 and expose port 80)
3. Initally run the Wallarm container (running on port 80, but exposed on port 81)
   - Using `6.3.0`, which is the version corresponding to latest on 20250709 @ 1600 ET
   - `docker run -d -e WALLARM_API_TOKEN='XXXXXXX' -e WALLARM_LABELS='group=<CDTGROUP>' -e NGINX_BACKEND='127.0.0.1' -e WALLARM_API_HOST='us1.api.wallarm.com' -p 81:80 wallarm/node:6.3.0`
4. Node deployed to Wallarm succesfully (using Docker Run and basic ENV vars versus the config file. This can provide additional options, but was not necessary for this exercise):
   - ![Node Deployed](images/node_deployed.png)
5. Tested using curl, and saw those items in the console
6. Tested via Postman, I see the sessions in the Wallarm console, but it was not successfully proxying the connection through
   - After some testing, realized I needed to use the *internal* IP of the httpbin server
   - Found the IP of both of the containers using `docker inspect | grep IPAddress`
     - `httpbin`: "IPAddress": "172.17.0.2",
     - `Wallarm Node`: "IPAddress": "172.17.0.3"
7. So, I redeployed the Wallarm Node container, using the internal (172.17.0.2) address for the API
    - - `docker run -d -e WALLARM_API_TOKEN='XXXXXXX' -e WALLARM_LABELS='group=<CDTGROUP>' -e NGINX_BACKEND='172.17.0.2' -e WALLARM_API_HOST='us1.api.wallarm.com' -p 81:80 wallarm/node:6.3.0`
8. Tested in Postman, and I got the result, and I see the session in Wallarm under `API Sessions`
    - ![Postman](images/postman_html.png)
    - ![API Sessions](images/api_sessions.png)

***Filtering Node and API backend running succesfully!***

### 3️⃣ Generate Traffic Using GoTestWAF
1. Using documentation here: https://github.com/wallarm/gotestwaf
2. I am using the Docker container for GoTestWAF
    - As of 20250709 @ 1700 ET, `latest` = `0.5.6`, so I will specify and use that
3. Ran the GoTestWAF tool, using this command:
    `docker run --rm --network="host" -it -v ${PWD}/reports:/app/reports wallarm/gotestwaf:0.5.6 --url="http://127.0.0.1:81" --noEmailReport`
    - Kept throwing error: `error="WAF was not detected. Please use the '--blockStatusCodes' or '--blockRegex' flags. Use '--help' for additional info. Baseline attack status code: 200"`
    - Reached out to Brandon, but in the meantime, I found several more options in the documentation
    - Added the `--skipWAFBlockCheck` flag, and the tool ran
4. Reports were generated, and these are added to my local directory, since I mapped the volume in the `docker run` command
5. I opened up the reports, and they look complete. I put these in the `reports/monitoring` folder
6. I also looked at the attacks in the Wallarm console, and they have increased significantly4
    - ![New Attacks](images/newattacks.png)

### Redeploy Wallarm in Blocking Mode
1. Stop the Wallarm Container and redploy it using the `-e WALLARM_MODE='block'` flag
    - `docker run -d -e WALLARM_API_TOKEN='XXXXXXX' -e WALLARM_LABELS='group=<CDTGROUP>' -e NGINX_BACKEND='172.17.0.2' -e WALLARM_API_HOST='us1.api.wallarm.com' -e WALLARM_MODE=block -p 81:80 wallarm/node:6.3.0`

**Important! Make sure you don't forget the label for the group. I left this off, and while traffic proxied through, it did not report into the mothership.**
- When this did happen, I looked at the Node in the Wallarm GUI, and it showed red and that it had not checked in recently, so that was the flag that something was not working properly.

2. Rerun the GoTestWAF tool, this time without the `--skipWAFBlockCheck` flag
    - It runs fine without that flag
3. I now see attacks have been blocked
    - ![Blocked Attacks](images/blocked.png)
4. Report went from an F to an A+!
    - These reports are under `reports/blocking`
    - ![Blocking Report Card](images/blocking_report.png)


### 4️⃣ Document Your Process
1. Progress, screenshots, troubleshooting, and steps have been documented in this README.md file

----------




---


## ✅ Evaluation Criteria

Your submission will be evaluated based on:

📌 **Completeness**: Were all required tasks completed?  
📌 **Clarity**: Is the documentation clear and well-structured?  
📌 **Troubleshooting**: How well did you document and resolve any issues?  
📌 **Understanding of the Product**: Did you correctly set up and use the Wallarm filtering node?  
📌 **Use of Official Documentation**: Did you successfully leverage Wallarm's official resources?  

---

## 📬 Submission

Once you have completed the evaluation, submit the following:

📂 Fork this **GitHub repo** and use it as the repository for your documentation, configuration files, and any relevant logs or screenshots.  
📜 A **README file** summarizing your process and key findings.  
📜 A **HIGH Level Diargram** that illustrates what you built and how traffic is flowing.  

---

## ℹ️ Additional Notes

💡 You are encouraged to **ask questions and leverage Wallarm's documentation**.  
📖 The ability to **document your troubleshooting steps** is just as important as the final deployment.  

🚀 **Good luck, and we look forward to your submission!** 🎉
