# easymirror-platform
All EasyMirror services started from a single repository with docker-compose.

## Building Containers
### Building
- To build all the services together, run `docker-compose up`
### Debugging
- To debug the docker containers, simply run `docker-compose config`

## Deployment
-Below are several options we considered when it came to deploying our production build. The ultimate goes was connecting the frontend and backend together under one domain, mainly for CORS related issues.
### Option 1
- The first that came to mind was deploying exatly how we deploy our local builds, with Docker. The benefit of that is that not only is it convient since it aligns with our dev build, but Docker would handle linking the containers.
- The problem with this method is that as of November 2023, it seems most, if not all, cloud providers have deprecated the use of docker-compose to upstream builds. That means we would have to "manually" link our containers. In AWS there were a few ways this could be done
    - Port Mapping
    - Using a ["bridge" network](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/networking-networkmode-bridge.html), which is the equivlant of docker-compose
    - Path based routing 

### Option 2
- Another option was hosting the frontend static files on AWS S3 and running backend app on AWS ECS.
- This method is one of the more cost effective since we would only be paying for a S3 bucket, with minimal files, and a ECS cluster.
- The problem with this falls on connecting the frontend and backend. We wouldn't be able to use an ALB for this approach, which is where CloudFront comes in.
### Option 3