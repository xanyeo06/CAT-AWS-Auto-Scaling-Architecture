# CAT — AWS Auto Scaling Architecture

The assignment required designing a highly available AWS network and deploying a
Flask web application that scales automatically under load. The build runs in
`us-east-1` across two Availability Zones, with `t2.micro` instances managed by an
Auto Scaling Group (min 2, max 3) under a target tracking policy on average CPU.
Scale-out was proven with a stress test that pushed the group from two instances to
three, plus supporting S3 and DynamoDB storage.

## Contents

| File | Description |
| --- | --- |
| `CAT-Assignment-1-Report.docx` | Full assignment report (19 pages) |

## Architecture

- **Region and VPC:** everything sits in `us-east-1` (N. Virginia) inside the default VPC, with one public subnet per Availability Zone (`us-east-1a`, `us-east-1b`). Losing a single zone leaves the application reachable through the other.
- **Internet gateway:** the single entry point for external traffic, routing requests on port 5000 to the Flask web servers.
- **Compute:** `t2.micro` instances (1 vCPU, 1 GB RAM), chosen to meet the assignment specification while staying Free Tier eligible, running Ubuntu Server 26.04 LTS (HVM). Access is by EC2 key pair.
- **Launch template:** `CAT-Asg1` bundles the AMI, instance type, security group and a User Data Bash script that updates packages, creates a Python virtual environment, installs dependencies and starts the Flask server, so every new instance boots configured.
- **Security group:** `CAT-Asg1-Web` follows least privilege, allowing only SSH (22), HTTP/HTTPS (80/443) and custom TCP 5000 for Flask.
- **Auto Scaling Group:** `CAT-Asg1-Auto_Scaling` uses the launch template, spans both AZs with a "balanced best effort" distribution, and holds desired 2, minimum 2, maximum 3.
- **Scaling policy:** target tracking on average CPU utilisation at 20%. Above the threshold the group adds a third instance; below it, capacity returns to two.

## AWS services used

| Service | Role in the solution |
| --- | --- |
| Amazon VPC | Network boundary, two public subnets across two AZs, route to the internet gateway |
| Amazon EC2 | `t2.micro` web servers running the Flask application |
| EC2 Launch Template | Standardised, repeatable instance configuration including User Data bootstrap |
| EC2 Auto Scaling Group | Maintains capacity, spreads instances across AZs, applies scaling policy |
| Amazon CloudWatch | CPU alarms that drive the target tracking policy |
| Security Groups | Stateful firewall restricting inbound traffic to ports 22, 80, 443 and 5000 |
| Amazon S3 (public bucket) | Brochure storage with ACLs enabled and public read on the uploaded PDF |
| Amazon S3 (internal bucket) | Private store for scaling evidence such as logs, screenshots and video |
| Amazon DynamoDB | `Products` table with a string partition key (`ID`) storing Category, Name, Price and Stock |

## Web server optimisation

The report explains why the Flask development server (Werkzeug) is unsuitable for production and sets out a two-tier alternative:

- **Gunicorn as WSGI gateway:** runs multiple worker processes to serve concurrent requests, monitors them and restarts any that fail, and translates raw HTTP into a form Flask can handle.
- **Nginx as reverse proxy:** receives public traffic on ports 80/443 and forwards it to Gunicorn on a protected internal port, serves static assets directly, and offloads TLS termination, load balancing and request buffering.
- **Division of labour:** Nginx handles the external interface, security and static content; Gunicorn provides the stable multi-process environment for application logic.

## Key takeaways

- A target tracking policy on average CPU at 20% reacted to a `stress --cpu 2 --timeout 300` load test without any manual step: the CloudWatch alarm entered `ALARM` and the group moved from 2 to 3 instances.
- The replacement instance launched into `us-east-1b`, and all three instances reached `Running` with 2/2 status checks passed, confirming the launch template and User Data script produce a working server every time.
- "Balanced best effort" placement kept capacity spread over both AZs, so the failure of one zone does not take the application down.
- Separating a public brochure bucket from a private evidence bucket shows data classification in practice, while DynamoDB's schemaless items suit a simple product catalogue without relational overhead.
- The exercise makes the gap between a development server and a production stack concrete: scaling compute is only half the problem, and the request path in front of the application matters just as much.

## Notes

- Credentials, key pairs, account identifiers and resource IDs (VPC, subnet, instance, public IPs) are not reproduced here; the original screenshots inside the report still show them.
- Section 5 of the report discusses Gunicorn and Nginx as the production architecture; the environment deployed and load tested for this assignment runs the Flask server directly on port 5000.
