# Phase 1 — Comparison Report

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->804.8<!--/answer--> | <!--answer:vm_warm_median_ms-->461.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2167.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->503.6<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->2027.9<!--/answer--> | <!--answer:function_warm_median_ms-->647.2<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
Warm latency distribution (ms), 20 samples per platform  [# = 1 sample]

IaaS VM  (min 419.3, median 461.4, p95 988.8, max 996.8)
       <450 | ######## 8
    450-500 | ##### 5
    500-550 | #### 4
    550-600 | 
    600-800 | 
   800-1000 | ### 3
  1000-1500 | 
     >=1500 | 

Cloud Run  (min 439.1, median 503.6, p95 812.4, max 819.3)
       <450 | # 1
    450-500 | ####### 7
    500-550 | ####### 7
    550-600 | # 1
    600-800 | ## 2
   800-1000 | ## 2
  1000-1500 | 
     >=1500 | 

Cloud Function  (min 483.4, median 647.2, p95 2359.3, max 2469.8)
       <450 | 
    450-500 | # 1
    500-550 | ##### 5
    550-600 | ### 3
    600-800 | ###### 6
   800-1000 | # 1
  1000-1500 | ## 2
     >=1500 | ## 2

```
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.**

<!--answer:scaling_behaviour-->
VM: nothing is created automatically. There is one e2-micro (shared vCPUs, 1 GB RAM) running one container, so the hard limit is the machine type I picked; beyond it requests queue or time out unless I add VMs by hand. Cloud Run: new container instances are created automatically as load grows and removed when idle, down to zero; the hard limit is my own flag `--max-instances=3`. Cloud Function: the same mechanism (it runs on Cloud Run), creating instances per load; I did not set a cap, so the limit is the platform default.
<!--/answer-->

**Deployment effort.**

<!--answer:deployment_effort-->
VM: the most work. I built a custom VPC, subnet and two firewall rules, created the VM with the right tag, granted its service account read access to Artifact Registry, accessed it via SSH, installed Docker, authenticated it and started the container; patching, restarts and HTTPS would also be mine. Cloud Run: build and push the image, then one deploy command; I also had to add the `allUsers` invoker role to make it public. Function: least effort, only the source and one command, since the platform builds the container itself, but I control the least (no Dockerfile, no base image choice).
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
   SERVICE   REGION       URL                                                LAST DEPLOYED BY                                              LAST DEPLOYED AT
✔  echo      us-central1  https://echo-722466236362.us-central1.run.app      service-722466236362@gcf-admin-robot.iam.gserviceaccount.com  2026-10-18T17:48:01.670806Z
✔  echo-bot  us-central1  https://echo-bot-722466236362.us-central1.run.app  paolo.malugani@eurecom.fr                                     2026-10-18T18:19:02.621395Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The listing shows my Function `echo` as a Cloud Run service, deployed by the Cloud Functions service agent (`gcf-admin-robot`) rather than by me. A gen2 function is therefore a Cloud Run service whose container Cloud Build assembles from my source with the Functions Framework. Both run on the same infrastructure, with the same scale-to-zero, image pull and process start, which is why their cold starts are close. What really differs is the contract: for the Function I hand over only source code (plus the runtime and entry point flags) and Google builds the container and serves it with the Functions Framework; for Cloud Run I hand over a finished image, so I control the Dockerfile, the base image, the server (gunicorn) and the port.
<!--/answer-->
