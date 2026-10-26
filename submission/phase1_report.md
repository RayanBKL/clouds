# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->50<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->echo-net<!--/answer-->

<!--answer:subnet_name-->echo-subnet<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->454.9<!--/answer--> | <!--answer:vm_warm_median_ms-->244.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1843.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->173.1<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1046.6<!--/answer--> | <!--answer:function_warm_median_ms-->177.5<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Warm-request latency (median, ms), measured from my laptop in France to us-central1:

```
vm        244.4 ms  ########################################
cloudrun  173.1 ms  #############################
function  177.5 ms  #############################
```

Cold start: VM 454.9 ms, Cloud Run 1843.7 ms, Function 1046.6 ms.
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
VM: nothing is created automatically. One e2-micro (2 shared vCPUs, 1 GB RAM) serves every request, so its hard limit is the size of the machine I picked with --machine-type=e2-micro, and a burst of 100 requests just queues on it. Cloud Run: it automatically creates extra container instances of my image as concurrent requests arrive, and removes them (down to zero) when idle; the hard limit is the --max-instances=3 flag I passed. Cloud Function: the platform also creates new instances on demand, up to the --max-instances=3 I set, and scales to zero when idle; each instance handles requests with the same concurrency model as Cloud Run since it runs on it.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
VM (most effort): I had to build a custom VPC, a subnet and two firewall rules (8080 and 22, matched by the echo-http network tag), create the instance, give its service account access to Artifact Registry, install Docker and authenticate it, and run the container. I also maintain the OS, Docker and security patches, and it keeps running and billing until deleted. Cloud Run: I only built the image (Dockerfile + Cloud Build + Artifact Registry) and ran one gcloud run deploy; networking, TLS, scaling and the servers are managed. Cloud Function (least effort): I deployed the source with one command, with no Dockerfile and no image to build or push; Google builds the container itself from my code and requirements.txt.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
   SERVICE   REGION       URL                                                LAST DEPLOYED BY                                              LAST DEPLOYED AT
✔  echo      us-central1  https://echo-811343710008.us-central1.run.app      service-811343710008@gcf-admin-robot.iam.gserviceaccount.com  2026-10-26T14:42:54.410610Z
✔  echo-bot  us-central1  https://echo-bot-811343710008.us-central1.run.app  rayan.bouakli@eurecom.fr                                      2026-10-26T14:41:31.310366Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output lists my Function "echo" as a Cloud Run service, next to "echo-bot", deployed by the Cloud Functions robot service account (gcf-admin-robot). So a gen 2 Cloud Function is not a separate runtime: it is a Cloud Run service whose container Google builds for me from my source with the Functions Framework. That is why its cold start resembles Cloud Run's: the same infrastructure has to schedule and start a container instance and boot Python after scaling to zero. What genuinely differs is what I deliver and who builds the container. With Cloud Run I ship my own image (my Dockerfile, gunicorn, 1 worker with 8 threads); with the Function I ship source and an entry point, and Google handles the image and server. The Function was somewhat faster to cold start here (1046.6 vs 1843.7 ms), probably because of the different base image and server and normal run-to-run variance, not a different platform.
<!--/answer-->
