# easymirror-platform
All EasyMirror services started from a single repository with docker-compose.

## Building Containers
### Building
- Ensure the [backend](https://github.com/easymirror/easymirror-backend) and [frontend](https://github.com/easymirror/easymirror-frontend) container images are built and deployed on Docker. Because we do not have Dockerhub, a local repository is fine.
- To build all the services together, run `docker-compose up -d`
- To shut down all services, run `docker-compose down`
### Debugging
- To debug the docker containers, simply run `docker-compose config`

## Deployment
-Below are several options we considered when it came to deploying our production build. The ultimate goes was connecting the frontend and backend together under one domain, mainly for CORS related issues.
### Option 1
- The first that came to mind was deploying exatly how we deploy our local builds, with Docker. The benefit of that is that not only is it convient since it aligns with our dev build, but Docker would handle linking the containers.
- The problem with this method is that as of November 2023, it seems most, if not all, cloud providers have deprecated the use of docker-compose to upstream builds. That means we would have to "manually" link our containers. In AWS there were a few ways this could be done:
    - Port Mapping
    - Using a ["bridge" network](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/networking-networkmode-bridge.html), which is the equivlant of docker-compose
    - Path based routing 
### Option 2
- Another option was hosting the frontend static files on AWS S3 and running backend app on AWS ECS.
- This method is one of the more cost effective since we would only be paying for a S3 bucket, with minimal files, and a ECS cluster.
- The problem with this falls on connecting the frontend and backend. We wouldn't be able to use an ALB for this approach, which is where CloudFront comes in.
### Our Choice
- Instead of deploying the nginx container to the cluster, we opted for an ALB that can also act as a reverse proxy. Not only is it able to act as a reverse proxy, but we get the added benfit of automatically scaling, according to this [response](https://www.reddit.com/r/devops/comments/10875aa/comment/j3qi2tn/?utm_source=share&utm_medium=web2x&context=3):
    > [AWS ALB] will automatically scale with the amount of traffic you're receiving and it's tightly integrated with the rest of the infrastructure.. Your autoscaling-group is growing, because of a traffic spike? No problem, the new nodes will automatically register with the loadbalancer..And it doesn't stop at single compute instances.. You can just as easily attach lambda, ECS or EKS...
- To replicate our local environment, we decided to go with AWS Fargate using "awsvpc" mode. By placing the container within the same task, we can communicate using "localhost:port number".

### Deploying on AWS
1. On [AWS ECR](https://us-east-1.console.aws.amazon.com/ecr/home?region=us-east-1), create a repository for the frontend and backend containers
    - Follow the `View push commands` command to upload to ECR.
2. Navigate to [AWS ECS](https://us-east-1.console.aws.amazon.com/ecs/v2/clusters?region=us-east-1) and create a new cluster
    - Under the `Infrastructure` dropdown, select `AWS Fargate (serverless)`
    - Leave everything else default
3. Create a task defintion
    - Choose `AWS Fargate` for Infrastructure requirements
    - CPU, Task roles, etc, all can be left default
    - When adding a container, use the Image URI from the AWS ECR
        - When adding a container, for the `Container port` section, make sure it is the same one that is becing exposed in the Dockerfile
            - Port `8080` = API container
            - Port `80` = Frontend container
4. Create 2 target groups
    - Under `EC2`, click on `target groups`
    - When choosing the target type, select `IP`
        - **Make sure to create IP target type, not instance. Fargete will not work with instance TGs**
    - In Step 2, after choosing target group details, remove the IPV4 address in the `Enter an IPv4 address from a VPC subnet.` input
    - Create 2 total target groups, one for frontend & api container
5. Create 2 Security Groups
    - Create a security group for the ALB that **only** allows connections from port 80 & 443
    - Create a security group for the service/containers, accepting inbound traffic from the load balancer
        - When selecting inbound rules, select the following config:
            - All TCP
            - Custom Source
            - Choose the security group that you made in the previous step
        - [Tutorial](https://youtu.be/rUgZNXKbsrY?si=vleV1j498v8cHMtr&t=165)
6. Create ALB
    - Under `EC2`, click on `load balancer`
    - Choose `Application Load Balancer` as the load balancer type
7. Go back to the task definition section under ECS:
    - Click on `Deploy` on the top right
    - Click on `Create Service`
    - Choose the cluster you created in step #2
    - **Make sure you have a public IP enabled**
    - Under the `Networking` tab, make sure you select the security group for the container and remove the default one
    - Under the `Load Balancing` tab, choose the load balancer created in step 6
#### Other Useful Resources

## Logs
- All logs will be stored in AWS' CloudWatch

## Notes
- This repository is meant to replicate the environment with a load balancer
- When it comes to SSL & port 443, because we are only accepting traffic form ALB, we do not need to have a nginx.conf file to listen on port 443.

## Other Resources
- [Route Traffic to Multiple Target Groups using Load Balancer Listener Rules ](https://www.youtube.com/watch?v=0XMsnAgHXoo)
- [Using ALB with Fargate](https://stackoverflow.com/questions/64096246/can-we-use-one-alb-with-aws-ecs-fargate)
- [Only Allow Traffic From Load Balancer](https://stackoverflow.com/questions/49227466/aws-instance-only-allow-traffic-from-load-balancer)
- [Application Load Balancer with ECS Fargate](https://stackoverflow.com/questions/64409699/application-load-balancer-with-ecs-fargate)

## TODOs
- [ ] Add cloudformation scripts to make creating services easier