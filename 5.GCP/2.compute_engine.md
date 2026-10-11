# Google Compute Engine: The Interview Cheat Sheet

Oct 2, 2026 · @Shubham

## 1. What is Compute Engine? (The 10-year-old version)

Compute Engine is Google renting you a computer over the internet, billed by the second.

Imagine you want to bake cakes for a living. You could build your own kitchen: buy ovens, fix the roof, pay the electrician. Or you could rent a ready-made kitchen, pay only for the hours you cook, and give it back when you are done. Compute Engine is that rented kitchen. The kitchen is a computer inside a Google data center, and you choose how many ovens it has (CPUs) and how much counter space (RAM).

**The grown-up version:** Compute Engine (GCE) is Google Cloud's IaaS (Infrastructure as a Service). It gives you virtual machines (VMs, also called instances) that run on Google's physical servers. You get full control of the operating system and everything above it. Google handles the buildings, power, cooling, hardware and hypervisor.

| Layer | Who looks after it |
| --- | --- |
| Data center, servers, hypervisor | Google |
| Physical network and physical security | Google |
| Guest OS, patches, runtime (for example the JVM), your app, your data | You |
| Firewall rules, IAM, VM sizing, backups | You |

**Your 20-second interview answer:** Compute Engine is Google Cloud's IaaS. It lets me create virtual machines with the CPU, memory, disk and OS I choose, scale them with managed instance groups, secure them with IAM and firewall rules, and pay per second. I use it when I need more control than Cloud Run or GKE give me.

**Words to remember:** IaaS, VM or instance, zonal resource, per-second billing, full control.

## 2. Why, When and Where to Use It

Use Compute Engine when you need control, and use something simpler when you do not.

**Why people pick it**

- You need a specific OS, kernel setting, agent, or licensed software.
- You are moving an existing app to the cloud without rewriting it (lift and shift).
- You need special hardware: GPUs, huge memory, fast local SSD.
- You run long-lived or stateful workloads such as self-managed databases or game servers.
- You want predictable performance and full network control.

**When NOT to pick it:** if your app is a stateless container, Cloud Run is simpler. If you have many microservices, GKE is better. If it is a small piece of event-driven code, use Cloud Run functions (formerly Cloud Functions). Every VM you run is a machine you must patch.

| Service | What you manage | Pick it when |
| --- | --- | --- |
| Compute Engine | VM, OS, runtime, app | Full control, legacy apps, special hardware |
| GKE | Containers and Kubernetes config | Many containerized microservices, orchestration |
| Cloud Run | Just a container | Stateless HTTP or jobs, scale to zero |
| App Engine | Just your code | Simple web apps, minimal ops |
| Cloud Run functions | Just a function | Small event-driven glue code |

**Where it lives: regions and zones**

Think of a region as a city (asia-south1 is Mumbai) and a zone as one separate building in that city (asia-south1-a). Each zone has its own power and cooling. A VM lives in exactly one zone, so if that zone has a bad day, a lone VM goes down with it.

| Scope | Examples |
| --- | --- |
| Zonal | VMs, zonal persistent disks, zonal managed instance groups |
| Regional | Subnets, static external IPs, regional persistent disks, regional MIGs |
| Global | VPC networks, firewall rules, images, snapshots, instance templates (global or regional), global load balancers |

**Interview tip:** when asked about high availability, start with: spread across zones, never rely on one VM.

## 3. The Building Blocks (Lego Pieces of a VM)

A VM is built from a handful of pieces. Know each one and you can answer most basic questions.

| Piece | Kid explanation | Technical note |
| --- | --- | --- |
| Instance | The computer itself | A VM in one zone, inside one project |
| Machine type | How strong the computer is | vCPU count plus memory, for example e2-medium |
| Image | The installed software on a USB stick | Public (Debian, Ubuntu, RHEL, Windows, Container-Optimized OS) or custom |
| Boot disk | The hard drive Windows or Linux starts from | Created from an image, usually a Persistent Disk or Hyperdisk |
| Extra disks | More drawers for your stuff | Data disks, Local SSD |
| VPC network and subnet | The road system the computer is plugged into | Gives the VM an internal IP |
| Service account | The computer's ID badge | Identity used to call Google APIs |
| Metadata | Sticky notes on the computer | Key-value data, startup scripts, SSH keys |
| Labels and network tags | Stickers | Labels organize and bill; tags attach firewall rules |

**Where VMs sit in the hierarchy:** Organization, then Folders, then Projects, then resources. IAM and quotas are applied at project level and above.

**Image vs template vs machine image (often confused)**

- **Image:** a copy of a disk used to create boot disks.
- **Instance template:** a saved recipe (machine type, image, network, labels) used to stamp out many VMs.
- **Machine image:** a full backup of one VM, including its config and all attached disks.

**Labels vs network tags:** labels are for cost tracking and filtering. Network tags are for firewall rules and routes. Mixing them up is a classic interview trap.

## 4. Machine Families and Types (Choosing the Size)

Choosing a machine type is like choosing a vehicle: a bicycle, a car, a truck, or a race car.

| Family | Example series | Best for |
| --- | --- | --- |
| General purpose | E2, N2, N2D, N4, C3, C4, Tau T2D | Web servers, apps, small to medium databases, dev and test |
| Compute optimized | C2, C2D, H3 | CPU-heavy work: HPC, gaming, video encoding, batch |
| Memory optimized | M-series (M1, M2, M3, M4) | SAP HANA, big in-memory databases and caches |
| Storage optimized | Z3 | High local SSD throughput and IOPS, big data, logging systems |
| Accelerator optimized | A-series, G-series (GPUs) | ML training and inference, rendering |

Arm-based options (Tau T2A, and newer Axion-based series) also exist for price-performance on Arm-compatible software.

**How to read a name:** n2-standard-4 means series N2, the standard memory ratio, 4 vCPUs.

- **standard:** about 4 GB RAM per vCPU
- **highmem:** about 8 GB per vCPU
- **highcpu:** about 1 GB per vCPU

**Other things to know**

- **vCPU** is one hardware thread (a hyperthread on most series), not a full physical core.
- **Shared-core types** (e2-micro, e2-small, e2-medium) can burst briefly. Good for dev boxes, bad for steady heavy load.
- **Custom machine types** let you pick exact vCPU and memory when predefined sizes do not fit. They are available on general-purpose families like N and E series, and cost a little more per unit than predefined.
- **Changing machine type** requires stopping the VM, editing the type, and starting it again. That is downtime.
- **Which to choose:** start with E2 or N2 for general work, watch Cloud Monitoring, then use rightsizing recommendations to move up or down.

**Worst case:** picking a giant machine for a small app (you pay for idle CPU), or a shared-core machine for a production database (CPU throttling).

## 5. Disks and Storage (Where the VM Keeps Its Stuff)

The VM is the desk. The disk is the drawer. Persistent Disk is a drawer in a locker room down the hall: it is connected over the network, and it survives even if you throw the desk away.

| Disk type | Speed | Survives VM stop or delete? | Use it for |
| --- | --- | --- | --- |
| pd-standard (HDD) | Slow | Yes | Cheap bulk or sequential data |
| pd-balanced | Good | Yes | Default for most workloads |
| pd-ssd | Fast | Yes | Databases, low latency |
| pd-extreme | Very high IOPS | Yes | Heavy databases (older option) |
| Hyperdisk (Balanced, Extreme, Throughput, ML) | IOPS and throughput tuned separately from size | Yes | Newer series such as C4 and N4, demanding databases |
| Local SSD | Fastest, attached to the host | No. Lost on stop, delete or preemption (survives reboot and live migration) | Cache, scratch, temporary data |

**Backups and copies**

- **Snapshot:** a backup of a disk. Incremental, global, and can be scheduled. You can restore it to a new disk in any zone.
- **Image:** a template for creating new boot disks.
- **Regional Persistent Disk:** copies data synchronously across two zones of one region. Used for HA databases.

**Rules interviewers love**

- You can **grow** a disk (even while the VM runs) but you can **never shrink** it. After growing, extend the partition and filesystem inside the OS.
- A boot disk is set to **auto-delete with the VM** by default. Data disks are not, unless you choose it.
- A **stopped VM still costs disk money.**
- One disk can be attached read-write to one VM, or read-only to many VMs.
- Data is **encrypted at rest by default**. You can bring your own keys with Cloud KMS (CMEK).
- For shared files across many VMs, use Filestore (NFS). For objects, use Cloud Storage.

**Best case:** pd-balanced for most things, scheduled snapshots, data separated from the boot disk.

**Worst case:** a production database on Local SSD with no replication or backup. One stop and it is gone.

## 6. Networking (How the VM Talks to the World)

A VPC is a private road network for your VMs. A firewall rule is a security guard at the gate holding a list of who may pass.

**VPC basics**

- A VPC network is **global**. Its subnets are **regional**.
- Use **custom-mode** VPCs in production. The auto-mode default network is fine for demos only.
- The default network comes with permissive rules (internal traffic, SSH, RDP, ICMP) that you should tighten.

**IP addresses**

| Type | Meaning | Notes |
| --- | --- | --- |
| Internal IP | Private address inside the VPC | Ephemeral by default, can be reserved |
| External IP | Public internet address | Ephemeral or static (reserved). External IPv4 addresses are billed, so avoid them when you can |

**Firewall rules**

- Applied at the VPC level, enforced at each VM. They are **stateful**.
- Each rule has direction (ingress or egress), action (allow or deny), priority, and a target.
- **Priority 0 is highest, 65535 lowest.** The default is 1000.
- **Implied rules:** deny all ingress, allow all egress, both at the lowest priority.
- Targets can be all instances, **network tags**, or **service accounts**. Service-account targeting is safer than tags.

**Reaching things safely**

| Need | Solution |
| --- | --- |
| Private VM needs outbound internet | Cloud NAT (outbound only, no inbound) |
| Private VM needs Google APIs | Private Google Access |
| SSH to a VM with no public IP | IAP TCP forwarding (allow source range 35.235.240.0/20 on port 22) |
| Connect to on-premises | Cloud VPN or Cloud Interconnect |
| Connect projects privately | VPC Peering or Shared VPC |
| Spread traffic across VMs | Cloud Load Balancing |

**Load balancer cheat line:** global external Application Load Balancer for HTTP(S) at layer 7, Network Load Balancer for TCP/UDP at layer 4, internal load balancers for traffic inside the VPC.

**Best case:** private IPs only, IAP for admin access, Cloud NAT for outbound, a load balancer as the only public door.

**Worst case:** a public IP on every VM with SSH open to 0.0.0.0/0.

## 7. Lifecycle, Startup Scripts, Access and Maintenance

**VM states and what you pay**

| State or action | What it means | What you still pay for |
| --- | --- | --- |
| Provisioning, Staging | Resources reserved, VM booting | Starting to bill |
| Running | Normal operation | vCPU, memory, disks, IPs, licenses |
| Stopped | Shut down gracefully | Disks, reserved static IPs. No vCPU or memory charge |
| Suspended | Memory state saved, VM paused | Storage for memory and disks |
| Reset | Hard reboot, like pressing the reset button | Same as running |
| Deleted | Gone | Nothing (except snapshots and images you keep) |

A VM can also show **Repairing** when Google is fixing a host problem.

**Startup and shutdown scripts**

- A **startup script** runs on every boot. Use it to install packages or pull config.
- A **shutdown script** is best effort and has only a short time window. Do not rely on it for critical saves.
- Keep scripts idempotent, meaning safe to run twice.
- Logs: run journalctl -u google-startup-scripts.service on Linux.

**Metadata server**

Every VM can ask for facts about itself at http://metadata.google.internal/computeMetadata/v1/ by sending the header Metadata-Flavor: Google. It returns instance name, zone, labels, custom attributes, and short-lived service account tokens. This is how apps get credentials without key files.

**Getting into a VM**

- Browser SSH from the console, or gcloud compute ssh.
- **OS Login (recommended):** SSH access is controlled by IAM roles (roles/compute.osLogin or osAdminLogin) and can require 2-step verification.
- **Metadata-based SSH keys:** older method where keys are stored in project or instance metadata. Harder to audit.
- **Serial console:** last resort when the VM cannot boot or networking is broken.
- Windows VMs use RDP after you reset the Windows password.

**Host maintenance**

Google updates hosts all the time. **Live migration** moves your running VM to another host with no reboot, so most VMs never notice. Two settings control behavior:

- **onHostMaintenance:** MIGRATE (default for most VMs) or TERMINATE (required for GPU VMs, Spot VMs and some others).
- **automaticRestart:** if the VM crashes or is terminated by Google (not by you), restart it.

**Moving a VM:** between zones or regions, snapshot the disks, create new disks in the target location, and build a new VM. Within a project, gcloud compute instances move can also do it for a stopped VM.

## 8. Pricing and Cost Control

You are billed **per second, with a one-minute minimum**. You pay for vCPU and memory, disks, external IPs, network egress, GPUs, and premium licenses (Windows, RHEL, SQL Server). Discount percentages below are approximate, so verify on the current pricing page before quoting exact numbers.

| Option | How it works | Approx. discount | Catch |
| --- | --- | --- | --- |
| On-demand | Pay list price, no promise | None | Most expensive for steady loads |
| Sustained use discount | Automatic discount when a VM runs most of the month | Up to about 30% | Applies to some series (such as N1, N2), not E2 |
| Committed use discount (CUD) | Promise to use resources for 1 or 3 years. Resource-based or flexible (spend-based) | Roughly 20% to 55% or more, higher for memory-optimized | You pay even if you stop using it |
| Spot VMs | Spare capacity at a big discount | Roughly 60% to 90% | Can be stopped at any time with about 30 seconds notice, no SLA, no live migration |
| Sole-tenant nodes | A physical server just for you | Premium price | For compliance, isolation, BYOL licenses |

**Spot vs preemptible:** preemptible is the older model with a hard 24-hour maximum runtime. Spot is the newer model with no fixed maximum runtime. Use Spot for new work.

**Cost-saving checklist**

- Rightsize using Recommender suggestions and Cloud Monitoring.
- Use CUDs for the steady baseline, Spot for fault-tolerant extras, on-demand for the rest.
- Use instance schedules to stop dev and test VMs at night and on weekends.
- Delete orphaned disks, old snapshots, and unused static IPs.
- Label everything, set budgets and alerts, and export billing data to BigQuery.
- Watch network egress, because data leaving Google or moving between regions costs money.

**Gotcha:** stopping a VM does not stop the bill for its disks or reserved IPs.

**Worst case:** a forgotten fleet of oversized VMs and unattached disks, discovered at month end.

## 9. Scaling and High Availability (One VM Is a Pet, Many Are Cattle)

A lone VM is like one worker with no backup. A managed instance group (MIG) is a manager who keeps a team of identical workers, hires more when busy, and replaces anyone who gets sick.

**The parts**

- **Instance template:** a frozen recipe. It is immutable. To change anything, create a new template and roll it out.
- **Managed instance group (MIG):** a set of identical VMs built from a template.
- **Autoscaler:** adds or removes VMs based on CPU utilization, load balancer capacity, Cloud Monitoring metrics, queue depth, or a schedule. You set minimum and maximum replicas and a cool-down period.
- **Autohealing:** a health check pings your app. If it fails, the MIG recreates the VM.
- **Rolling updates:** swap old VMs for new ones gradually. Control the pace with maxSurge and maxUnavailable. You can run a canary by giving a small share of VMs the new template.
- **Load balancer:** sends traffic only to healthy VMs.

**Zonal vs regional MIG:** a zonal MIG lives in one zone. A **regional MIG spreads VMs across up to three zones**, so losing one zone does not take you down. Use regional for production.

**Other group types:** a **stateful MIG** keeps per-VM disks, names or IPs through recreation (useful for databases and legacy apps). An **unmanaged instance group** is just a bag of different VMs. It has no autoscaling or autohealing.

**The HA ladder**

| Level | Setup | Survives |
| --- | --- | --- |
| 1 | Single VM with auto-restart and live migration | Host maintenance and host failures |
| 2 | Single VM with regional persistent disk | Zone data loss, with manual failover |
| 3 | Regional MIG behind a load balancer | A full zone outage |
| 4 | Multi-region with global load balancer and replicated data | A regional outage |

Compute Engine's published SLA is highest for instances spread across multiple zones, and much lower for a single instance. Check the current SLA page for exact figures.

**Classic mistakes**

- Health check **initial delay too short** for a slow-starting app, so the MIG kills VMs in a loop.
- Forgetting to allow Google health check ranges (130.211.0.0/22 and 35.191.0.0/16) in the firewall.
- Storing session data or files on the VM's local disk, which breaks when VMs are replaced. Keep state in Cloud SQL, Memorystore, or Cloud Storage.

## 10. Security (Locks, Badges and Guards)

Think in layers: who can touch the VM, what the VM can touch, how traffic reaches it, and how the VM itself is hardened.

| Layer | What to do |
| --- | --- |
| IAM (who manages VMs) | Least privilege. Common roles: compute.admin, compute.instanceAdmin.v1, compute.viewer, compute.osLogin, iam.serviceAccountUser (needed to attach a service account to a VM) |
| Service account (what the VM can do) | Create a **dedicated service account per workload** with only the roles needed. The default Compute Engine service account has the broad Editor role, which is a known bad practice |
| Credentials | Do not put JSON key files on VMs. Use the attached service account and the metadata server |
| Network | No external IP, IAP for admin access, Cloud NAT for outbound, tight firewall rules, Private Google Access |
| OS access | OS Login with 2-step verification, block project-wide SSH keys, disable the serial port |
| VM hardening | **Shielded VM** (secure boot, vTPM, integrity monitoring). **Confidential VM** (memory encrypted while in use) |
| Patching | OS patch management through VM Manager, or rebuild golden images and roll them out |
| Data | Encrypted at rest by default. Use CMEK for key control. Restrict who can read snapshots and images |
| Guardrails | Organization policies such as restricting external IPs, requiring Shielded VM, requiring OS Login |
| Audit | Cloud Audit Logs (Admin Activity logs are always on), plus Ops Agent for OS and app logs |

**Shielded vs Confidential in one line:** Shielded VM protects the **boot process** from tampering. Confidential VM protects **data in memory** from the host.

**Best case:** private VMs, per-app service accounts, OS Login, org policies, signed hardened images.

**Worst case:** default service account with Editor, public IPs, SSH open to the world, key files copied onto the VM.

## 11. Production Scenarios (Tell These as Stories)

Use this pattern in interviews: situation, design, why Compute Engine, trade-off.

**Scenario 1: Three-tier web application**

- **Design:** global external Application Load Balancer, then a regional MIG running a Java Spring Boot API from a custom image, then Cloud SQL for the database.
- **Details:** private IPs, Cloud NAT, IAP for admin access, autoscaling on CPU, rolling updates for releases, health checks for autohealing, logs and metrics through Ops Agent.
- **Why GCE:** custom runtime and agents, predictable performance.
- **Trade-off:** you patch the OS. Cloud Run might be cheaper if the app is fully containerized and stateless.

**Scenario 2: Nightly batch or ETL job**

- **Design:** MIG of Spot VMs reading jobs from a queue, writing results and checkpoints to Cloud Storage.
- **Why:** up to about 60 to 90% cheaper, and the work can be retried.
- **Trade-off:** VMs can vanish, so every task must be idempotent and resumable.

**Scenario 3: Lift and shift from on-premises**

- **Design:** migrate VMs with Migrate to Virtual Machines, keep IPs and hostnames where needed, connect with Cloud VPN or Interconnect.
- **Next step:** rightsize after a few weeks of real metrics, then modernize piece by piece.
- **Trade-off:** fast migration, but you carry over old inefficiencies.

**Scenario 4: Licensed or compliance-heavy app**

- **Design:** sole-tenant nodes for BYOL licenses or isolation, custom machine types to match license core counts, regional disks for resilience.

**Scenario 5: GPU model training**

- **Design:** accelerator-optimized VMs (A or G series), data in Cloud Storage, checkpoints saved regularly, Spot where interruption is acceptable.
- **Alternative:** Vertex AI if you want managed training.

**Scenario 6: Self-managed database**

- **Design:** memory-optimized or high-IOPS VM, SSD or Hyperdisk, regional disk or replica in a second zone, snapshot schedule, tested restores.
- **Honest note:** prefer Cloud SQL, AlloyDB or Spanner unless you need OS-level control.

**Scenario 7: Dev and test fleet**

- **Design:** small E2 VMs, instance schedules that stop them at night, labels per team, budget alerts.

**Release pattern:** bake an image (Packer or Image Builder), define it as code with Terraform, build a new template, roll it out with a rolling update or blue-green with two MIGs.

## 12. Best Cases and Worst Cases

**Where Compute Engine shines (best cases)**

- Steady, long-running workloads where CUDs cut the cost.
- Apps that need a custom OS, agents, drivers or licensed software.
- Stateless web or API tiers in a regional MIG behind a load balancer.
- Lift and shift migrations that must happen fast.
- GPU, high-memory or high-IOPS jobs.
- Fault-tolerant batch work on Spot VMs.
- Everything defined as code (Terraform), built from golden images, observable through Cloud Monitoring.

**Where it hurts (worst cases)**

- A single hand-configured VM nobody can rebuild (a pet). If it dies, so does the business.
- Running everything in one zone.
- Spot VMs for a primary database or anything that cannot be interrupted.
- Default service account with Editor, public IPs, SSH open to the internet.
- No snapshots, or snapshots that were never test-restored.
- Over-provisioned machines and forgotten disks, IPs and snapshots.
- Treating VMs like on-premises servers: manual patching, no automation, no monitoring.
- Storing important data only on Local SSD.

**Limitations to name honestly**

- You own OS patching and security above the hypervisor.
- VMs are zonal. Zone capacity can run out (stockout), so have fallbacks.
- Quotas limit CPUs, IPs and disks per region. Ask for increases early.
- Disks cannot shrink. Machine type changes need a stop.
- It is more work than serverless options.

**Pets vs cattle:** a pet VM has a name, a history, and manual fixes. Cattle VMs are identical, disposable and rebuilt from an image. Production should be cattle.

## 13. Troubleshooting Guide (Symptom, Cause, Fix)

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Cannot SSH | Firewall blocks port 22, no external IP and not using IAP, missing OS Login role, sshd down, disk full | Allow port 22 (or 35.235.240.0/20 for IAP), use --tunnel-through-iap, grant roles/compute.osLogin, check the serial console |
| VM will not boot | Corrupt boot disk, bad fstab, full disk, kernel issue | Read serial port output, attach the disk to a rescue VM, fix and reattach |
| QUOTA\_EXCEEDED error | Regional quota for CPUs, IPs or disk is used up | Request a quota increase, or use another region |
| ZONE\_RESOURCE\_POOL\_EXHAUSTED | The zone is out of that machine type | Try another zone or machine type, use a regional MIG, or reserve capacity in advance |
| Disk full | Logs or data filled the disk | Resize the disk, then grow the partition and filesystem (growpart, resize2fs or xfs\_growfs) |
| App slow, high CPU | Undersized VM, shared-core throttling, disk IOPS limits, noisy workload | Check Cloud Monitoring and Ops Agent metrics, rightsize, use a faster disk type or larger disk |
| Startup script did not work | Wrong metadata key, script error, no network yet | Read the startup script logs, make the script idempotent, test it by hand |
| MIG keeps recreating VMs | Failing health check, initial delay too short, health check ranges blocked | Fix the app endpoint, raise the initial delay, allow Google health check ranges |
| Private VM cannot reach the internet | No external IP and no NAT | Add Cloud NAT, or Private Google Access for Google APIs only |
| Cannot create VM with a service account | Missing iam.serviceAccountUser | Grant that role on the service account |
| Spot VM disappeared | It was preempted | Expected behavior. Add checkpoints, automatic restart logic and a fallback |
| Surprise bill | Public IPs, orphaned disks, snapshots, egress, forgotten VMs | Billing export to BigQuery, labels, budgets, cleanup scripts |

**Golden debugging order:** 1. Is the VM running? 2. Read the serial port output. 3. Check firewall and routes. 4. Check IAM and service account. 5. Check logs and metrics. 6. Check quotas.

## 14. gcloud Command Cheat Sheet

```bash
# Defaults
gcloud config set project my-project
gcloud config set compute/zone asia-south1-a

# Create a VM (use a current image family)
gcloud compute instances create web-1 \
  --machine-type=e2-medium \
  --image-family=debian-12 --image-project=debian-cloud \
  --boot-disk-size=20GB --boot-disk-type=pd-balanced \
  --tags=http-server --no-address \
  --service-account=app-sa@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --metadata-from-file=startup-script=startup.sh

# Spot VM
gcloud compute instances create batch-1 --provisioning-model=SPOT \
  --instance-termination-action=STOP --machine-type=n2-standard-4

# Daily operations
gcloud compute instances list
gcloud compute instances describe web-1
gcloud compute instances stop web-1
gcloud compute instances start web-1
gcloud compute instances reset web-1
gcloud compute instances delete web-1

# SSH (with and without a public IP)
gcloud compute ssh web-1
gcloud compute ssh web-1 --tunnel-through-iap

# Change machine type (VM must be stopped)
gcloud compute instances stop web-1
gcloud compute instances set-machine-type web-1 --machine-type=n2-standard-4
gcloud compute instances start web-1

# Debug
gcloud compute instances get-serial-port-output web-1

# Disks, snapshots, images
gcloud compute disks create data-1 --size=100GB --type=pd-ssd
gcloud compute instances attach-disk web-1 --disk=data-1
gcloud compute disks resize data-1 --size=200GB
gcloud compute disks snapshot data-1 --snapshot-names=data-1-snap
gcloud compute images create web-img --source-disk=web-1 --source-disk-zone=asia-south1-a

# Static IP and firewall
gcloud compute addresses create web-ip --region=asia-south1
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80 --target-tags=http-server --source-ranges=0.0.0.0/0

# Template, MIG, autoscaling, rolling update
gcloud compute instance-templates create web-tpl-v1 \
  --machine-type=e2-medium --image-family=debian-12 --image-project=debian-cloud
gcloud compute instance-groups managed create web-mig \
  --template=web-tpl-v1 --size=2 --region=asia-south1
gcloud compute instance-groups managed set-autoscaling web-mig \
  --region=asia-south1 --min-num-replicas=2 --max-num-replicas=6 \
  --target-cpu-utilization=0.6 --cool-down-period=90
gcloud compute instance-groups managed rolling-action start-update web-mig \
  --region=asia-south1 --version=template=web-tpl-v2 --max-surge=1 --max-unavailable=0

# Move a stopped VM to another zone
gcloud compute instances move web-1 --zone=asia-south1-a --destination-zone=asia-south1-b
```

Flags change over time, so confirm with gcloud compute instances create --help before relying on any of them in a live system.

## 15. Interview Rapid-Fire Q&A

**Basics**

- **What is Compute Engine?** Google Cloud's IaaS. Virtual machines with full control of OS and software, billed per second.
- **Is a VM zonal or regional?** Zonal. Subnets and static IPs are regional. VPCs, firewall rules, images and snapshots are global.
- **Machine type vs image vs template?** Machine type is the size, image is the software, template is the saved recipe combining them.
- **Compute Engine vs GKE vs Cloud Run?** GCE gives VMs and full control. GKE orchestrates containers. Cloud Run runs stateless containers with no servers to manage.
- **Labels vs network tags?** Labels are for billing and organization. Network tags are for firewall rules and routes.
- **What is a vCPU?** A hardware thread, usually one hyperthread of a physical core.

**Operations**

- **Can you change machine type on a running VM?** No. Stop it, change the type, start it.
- **Can you shrink a disk?** No. Create a smaller disk and copy the data. You can grow a disk online.
- **How do you move a VM to another zone or region?** Snapshot or image the disks, create disks and a VM in the target location. Or use instances move for a stopped VM in the same project.
- **Snapshot vs image vs machine image?** Snapshot backs up a disk. Image creates boot disks. Machine image backs up the whole VM config plus disks.
- **Persistent Disk vs Local SSD?** Persistent Disk is network-attached and durable. Local SSD is fast but ephemeral, lost on stop or delete.
- **What is live migration?** Google moves a running VM to another host during maintenance without a reboot. Not available for GPU or Spot VMs.
- **Stop vs suspend vs delete?** Stop frees CPU and RAM billing but keeps disk billing. Suspend saves memory state. Delete removes the VM.
- **How do you keep the same public IP?** Reserve a static external IP and attach it.
- **What is a startup script?** Code in metadata that runs on every boot, used for configuration.
- **What is the metadata server?** An internal endpoint where a VM reads its own details and service account tokens.

**Scaling and HA**

- **What is a MIG?** A group of identical VMs made from one template, with autoscaling, autohealing, rolling updates and multi-zone support.
- **Can you edit an instance template?** No, it is immutable. Create a new one and roll it out.
- **What can autoscaling use as a signal?** CPU, load balancer utilization, Cloud Monitoring metrics, queue backlog, schedules.
- **Autohealing vs autoscaling?** Autohealing replaces unhealthy VMs. Autoscaling changes the number of VMs.
- **How do you design a highly available app on GCE?** Regional MIG across zones, global load balancer with health checks, stateless VMs, managed or replicated data tier, and tested backups.
- **Zonal vs regional MIG?** Regional spreads VMs across zones and survives a zone outage.
- **How do you do zero-downtime releases?** Rolling update with maxUnavailable of 0, or blue-green with two MIGs.

**Cost**

- **Sustained use vs committed use discounts?** Sustained is automatic for steady use on some series. Committed is a 1 or 3 year promise for a bigger discount.
- **Spot vs preemptible?** Both are discounted spare capacity that can be stopped. Spot has no 24-hour limit and is the current model.
- **Top ways to cut cost?** Rightsize, CUDs for baseline, Spot for fault-tolerant work, schedules for dev VMs, cleanup of orphaned resources, E2 machines where suitable.

**Networking and security**

- **How do you SSH without a public IP?** IAP TCP forwarding with gcloud compute ssh --tunnel-through-iap, and allow 35.235.240.0/20 on port 22.
- **How does a private VM reach the internet?** Cloud NAT. For Google APIs only, Private Google Access.
- **How do firewall rules prioritize?** Lower number wins. Default 1000. Implied deny ingress and allow egress at 65535.
- **Why avoid the default service account?** It has Editor access, far more than most workloads need. Use a dedicated least-privilege service account.
- **How does a VM access Cloud Storage securely?** Attach a service account with a role such as storage.objectViewer. No key files.
- **Shielded VM vs Confidential VM?** Shielded protects boot integrity. Confidential encrypts memory in use.
- **What is a sole-tenant node?** A physical host dedicated to your project, for isolation, compliance or BYOL licensing.
- **How do you patch a fleet?** VM Manager patch management, or bake a new image and roll it out through the MIG.

**Troubleshooting**

- **VM creation fails with quota or resource errors?** Quota means request an increase. Resource pool exhausted means try another zone or reserve capacity.
- **Where do you look when a VM will not boot?** Serial port output, then rescue by attaching the boot disk to another VM.
- **MIG keeps recreating VMs?** Check health check path, initial delay, and firewall rules for health check ranges.

**ACE exam crossovers worth revising:** IAM roles and service accounts, gcloud config and configurations, billing and budgets, snapshot schedules, instance schedules, firewall rules, load balancer types, and Cloud Monitoring alerts.

## 16. Last-Minute Revision Sheet

**Memory tricks**

- **Scope: Z, R, G.** Zonal: VMs and zonal disks. Regional: subnets, static IPs, regional disks. Global: VPC, firewall, images, snapshots.
- **Stop is not free:** you still pay for disks and reserved IPs.
- **Template is frozen:** new version means new template.
- **Spot can vanish:** cheap, but never for the only copy of anything.
- **Private is the default:** no public IP, IAP in, NAT out.
- **Badge, not key:** use the attached service account, never key files.
- **Cattle, not pets:** rebuild from an image instead of fixing by hand.

**Your 60-second story structure**

1. What the workload was and why it needed VMs.
2. The design: regional MIG, load balancer, private networking, managed data tier.
3. How you handled security, scaling and cost.
4. One failure or trade-off you faced and what you changed.
5. What you would do differently, such as moving to Cloud Run for the stateless parts.

**Checklist before the interview**

- [ ] I can explain IaaS and the shared responsibility split.
- [ ] I know the machine families and how to read n2-standard-4.
- [ ] I can list disk types and say which survive a stop.
- [ ] I can explain snapshot, image, machine image and template.
- [ ] I know firewall priorities and implied rules.
- [ ] I can explain IAP, Cloud NAT and Private Google Access.
- [ ] I can design a regional MIG with autoscaling, autohealing and a load balancer.
- [ ] I can compare on-demand, sustained use, committed use and Spot.
- [ ] I can explain OS Login, Shielded VM, Confidential VM and service account best practice.
- [ ] I have two production stories ready: one success and one failure.
- [ ] I can say when I would NOT use Compute Engine.

**Final mindset:** interviewers rarely want memorized definitions. They want to hear trade-offs: why this choice, what it costs, how it fails, and how you would fix it.
