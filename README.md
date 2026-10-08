# AWS Cloud Support Lab — Monitoring & Incident Response on EC2

A hands-on lab that simulates two production-style incidents on AWS, detects them with CloudWatch, investigates them from the Linux command line, and documents each one as a postmortem — the same workflow a cloud support engineer follows day to day.

**Author:** Piyush Awasthi

---

## What I built

- **Compute:** one EC2 instance (`support-project-web01`, `t3.micro`, Amazon Linux) in `us-east-1`
- **Application:** Apache HTTP Server (`httpd`) serving a simple status page
- **Monitoring:** two CloudWatch alarms
  - `web01-high-cpu` — `CPUUtilization > 70` for 3 datapoints within 15 minutes
  - `web01-status-check-failed` — EC2 status check failures
- **Failure injection:** the `stress` utility (CPU load) and `systemctl stop httpd` (service outage)

---

## Environment setup

**EC2 instance running** (t3.micro, public IPv4 assigned, state: Running)

![EC2 instance summary](screenshots/setup-1-ec2-instance.png)

**Web server serving the baseline page**

![Baseline web page](screenshots/setup-2-web-page.png)

**Apache service healthy** (`active (running)`, listening on port 80)

![systemctl status httpd showing active](screenshots/setup-3-httpd-running.png)

**CloudWatch alarms configured and in OK state**

![CloudWatch alarms list](screenshots/setup-4-alarms.png)

---

## Incident reports

| # | Incident | Failure type | Key finding | Report |
|---|---|---|---|---|
| 1 | CPU spike | Resource exhaustion | Short bursts hit ~100% CPU but never fired the alarm; only a sustained run satisfied its 3-of-3 rule, and detection took ~14 minutes | [01-cpu-spike-postmortem.md](01-cpu-spike-postmortem.md) |
| 2 | Service outage | Application failure | Instance health checks stayed green while the website was down; `ERR_CONNECTION_REFUSED` plus a clean exit status pointed straight to a stopped service | [02-service-outage-postmortem.md](02-service-outage-postmortem.md) |

---

## What I learned

- **An alarm that looks correct can still miss real problems.** The CPU alarm worked exactly as configured, but its evaluation window didn't match the short bursts I was testing with. Alarm thresholds and evaluation periods have to match the failure pattern being monitored, and they set a floor on how quickly you can detect it.
- **A healthy instance does not mean a healthy application.** EC2 status checks and CPU alarms were both normal while the site was completely down. Real monitoring needs an application-level check as well as infrastructure metrics.
- **Error messages are evidence.** A refused connection (as opposed to a timeout) and a clean exit status in `systemctl status` narrowed the cause to the application layer in seconds, without touching networking or security groups.

---

## Skills demonstrated

AWS EC2 · Amazon CloudWatch (metrics and alarms) · Linux troubleshooting (`systemctl`, `ps`, `top`) · Apache httpd · SSH · root-cause analysis · incident timelines and postmortem writing# aws-incident-response-lab
