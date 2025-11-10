#  Level: Cloud Networking & Deployment
*Project: Build a 2-Tier App with VPC, Private DB & Public App Layer
Design and deploy a secure 2-tier web app on AWS: a public app layer (Nginx + sample app) talking to a private database (PostgreSQL/RDS). You’ll practice real cloud networking (subnets, routing, security), least-privilege access, and basic resilience.*

## Tasks:

- Create a custom VPC with public and private subnets.
- Deploy a PostgreSQL DB in a private subnet.
- Deploy an Nginx + sample app in a public subnet.
- Configure security groups & NACLs to allow only app → DB traffic.


## Deliverable:
- Architecture diagram + working app + secure VPC design.
- Ensure services on each subnet can communicate. 
- You are to pick any app of your choice that requires a DB communication.
- Your work is marked as successful when your frontend on a public subnet is able to communicate with your backend on a private subnet. Eg, a registration form that submits to a DB successfully  
- You can use any cloud of your choice - AWS, Azure, GCP