\# Stage 2 — Phase 1: VPC Foundation



\## What I Built



A production-grade 3-tier VPC on AWS in the `ap-south-1` (Mumbai) region.



\## Architecture

VPC: 10.0.0.0/16 (vpc-067648cd5c3f3f2ad)



┌─────────────────────────────────────────────────┐

│ Public Subnets (internet via IGW) │

│ ┌──────────────┐ ┌──────────────┐ │

│ │ public-1a │ │ public-1b │ │

│ │ 10.0.1.0/24 │ │ 10.0.2.0/24 │ │

│ │ (AZ-a) │ │ (AZ-b) │ │

│ └──────────────┘ └──────────────┘ │

│ │ │

│ │ Internet Gateway (igw-3tier) │

└─────────┼───────────────────────────────────────┘

│

┌─────────┼───────────────────────────────────────┐

│ App Subnets (internet OUTBOUND ONLY via NAT) │

│ ┌──────────────┐ ┌──────────────┐ │

│ │ app-1a │ │ app-1b │ │

│ │ 10.0.3.0/24 │ │ 10.0.4.0/24 │ │

│ └──────────────┘ └──────────────┘ │

│ │ │

│ └── NAT Instance (i-081db7405c4d181f6) │

└─────────────────────────────────────────────────┘

│

┌─────────┼───────────────────────────────────────┐

│ Data Subnets (fully isolated — NO internet) │

│ ┌──────────────┐ ┌──────────────┐ │

│ │ data-1a │ │ data-1b │ │

│ │ 10.0.5.0/24 │ │ 10.0.6.0/24 │ │

│ └──────────────┘ └──────────────┘ │

└─────────────────────────────────────────────────┘



\## Route Tables



| Route Table | Attached To | Internet Route | Purpose |

|:---|:---|:---|:---|

| `public-rt` | public-1a, public-1b | `0.0.0.0/0` → IGW | Load balancer + NAT |

| `app-rt` | app-1a, app-1b | `0.0.0.0/0` → NAT instance | App servers (outbound only) |

| `data-rt` | data-1a, data-1b | ❌ none | Database (isolated) |



\## Key Design Decisions



\### Why NAT Instance instead of NAT Gateway?

\- \*\*NAT Gateway\*\*: \~$32/month, fully managed

\- \*\*NAT Instance\*\* (t3.micro): \~$4/month, self-managed

\- \*\*Trade-off\*\*: Save 90% cost; acceptable for learning and small-scale projects

\- \*\*Interview answer\*\*: \*"I chose a NAT instance because the NAT Gateway is idle expensive at \~$32/month. A t3.micro with iptables MASQUERADE gives the same functionality at a fraction of the cost. For production with heavy traffic or compliance needs, I'd use a managed NAT Gateway."\*



\### Why 3 separate route tables?

\- \*\*Isolation\*\*: Each tier has different internet access needs

\- \*\*Security\*\*: Data tier has NO internet route → DB can never be accessed from outside

\- \*\*Blast radius\*\*: Compromise in one tier can't pivot through another's network



\### Why 2 AZs for each tier?

\- \*\*High Availability\*\*: If AZ-a fails, AZ-b keeps serving

\- \*\*No single point of failure\*\*: ALB distributes traffic across both

\- \*\*Future-proofing\*\*: Ready for RDS Multi-AZ when budget allows



\## Components Created



| Resource | ID | Cost |

|:---|:---|:---|

| VPC | vpc-067648cd5c3f3f2ad | $0 |

| 6 subnets | — | $0 |

| Internet Gateway | igw-069688acb21529b3a | $0 |

| NAT Instance | i-081db7405c4d181f6 | \~$0.012/hr |

| public-rt | rtb-0bc0446b8431ed92f | $0 |

| app-rt | rtb-0c85652f0fdde8d22 | $0 |

| data-rt | rtb-0b0279a9984460303 | $0 |

| NAT Security Group | sg-0b0ed61072782ef54 | $0 |



\## Troubleshooting



\### Issue: NAT instance launched without SSH key

\*\*Cause:\*\* Forgot `--key-name` in `run-instances` command.

\*\*Fix:\*\* Terminated and relaunched with `--key-name url-shortener-key`.

\*\*Lesson:\*\* Always include all required parameters. AWS CLI won't always warn if a parameter is missing — it just creates a broken resource.



\### Issue: iptables MASQUERADE rule missing

\*\*Cause:\*\* User-data script ran `iptables` before installing `iptables-services`.

\*\*Fix:\*\* Installed package first, then added rule manually, then saved.

\*\*Lesson:\*\* Order matters in user-data. Install dependencies before using their commands. Test user-data on a fresh instance.



\### Issue: SSH "Permission denied" on Windows

\*\*Cause:\*\* Windows file permissions on `.pem` were too open.

\*\*Fix:\*\* `icacls` to strip inherited permissions, granted only current user read access.

\*\*Lesson:\*\* On Windows, always `icacls file.pem /inheritance:r` before SSH.



\## Cost Optimization



| Item | Cost |

|:---|:---:|

| NAT instance while running | \~$0.017/hr |

| VPC, subnets, IGW, routes | \*\*$0\*\* |

| \*\*Overnight (NAT stopped)\*\* | \*\*\~$0.02/day\*\* |



\*\*Habit:\*\* Stop NAT instance at the end of every work session. Restart next session — route tables keep pointing to the instance ID, so no reconfiguration needed.



\## What's Next



\- \*\*Phase 2\*\*: Application Load Balancer + Auto Scaling Group

\- \*\*Phase 3\*\*: RDS + Secrets Manager

\- \*\*Phase 4\*\*: Codify everything in Terraform

