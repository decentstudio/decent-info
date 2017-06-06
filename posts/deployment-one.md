# Our First Deployment

## Summary

The end result is a single node Docker Swarm deployment with several services and one overlay network. Not suitable for a production deployment or even a multi-node staging/development environment due to some limitations with digital ocean.

Services:
- Nginx + CertBot
  - letsencrypt certificate
  - `in.decentstudio.com` subdomain
- RabbitMQ
- decent-slack-receiver
  - Behind nginx
- decent-news-distributor
  - Listening on 8081, not behind nginx

Overlay Network:
- decent-news-net

Digital Ocean Volume
- 1GB

### 1. Provisioning With Docker Machine

This part was very straightforward. I followed the directions in the Docker Machine tutorial. The most involved part is attaching some persistent storage.

DigitalOcean Driver Reference:

https://docs.docker.com/machine/drivers/digital-ocean/

1. Create an API key on digitalocean.com
2. Setup some env vars used by the digitalocean driver for convenience
   - DIGITALOCEAN_ACCESS_TOKEN
     - This is your API key
   - DIGITALOCEAN_REGION
     - I used nyc1 because of volumes
   - DIGITALOCEAN_PRIVATE_NETWORKING
     - This is for swarm node communication
3. Execute `docker-machine create --driver digitalocean decent-swarm-1`
4. The driver doesn't have an option to attach a volume yet.
   - Go create a persistent volume on digitalocean.com
   - Attach it to the `decent-swarm-1` droplet
   - `docker-machine ssh decent-swarm-1`
   - Follow the attachment instructions from digitalocean.com

### 2. Initialize The Docker Swarm

This part is very easy. I initially setup a 2 node swarm, but for reasons that will be explained later, I dropped it down to a 1 node swarm to get the dev deployment up ASAP. I followed the Docker Swarm instructions.

Docker Swarm Init Reference:

https://docs.docker.com/engine/reference/commandline/swarm_init/

1. Find the private IP for decent-swarm-1 on digitalocean.com
1. Execute `docker-machine ssh decent-swarm-1`
2. Execute `docker swarm init --advertise-addr <private-ip>`
3. We would initialize any more nodes here if we needed to using the command `init` provides us.

### 3. Create The Overlay network

This part is very easy. The overlay network is special to Swarm deployments. It allows container to container communication across Swarm nodes. We will be adding each of our services to this network later on.

1. From decent-swarm-1 execute `docker network create --driver overlay decent-news-net`

### 4. Glossing Over Some Things

I'm leaving out details about setting up DNS forwarding for `in.decentstudio.com` on google domains and creating our docker hub account `decentstudio`, creating docker hub repositories, and pushing the images. These subjects may be covered in an appendix at the end of this document.

### 5. Create The NGINX Service

This part was not straightforward at all and was the greatest challenge. It is easy to set up an NGINX container for http, but it is more involved to figure out how to get the certificate and how to configure the container to use https. It was at this step that I eventually saw the need to drop to one node in the Swarm. This is because the nginx-certbot container I was using for convenience required a volume be mounted. The persistent volumes provided by digitalocean do not support being attached to multiple droplets. Therefore, instead of trying to figure out how to use a networked storage system, I dropped the deployment down to one node.

The command that finally worked:

```
docker service create                                                                                                   \
--name=<service-name>                                                                                                   \
--network=<network-name>                                                                                                \
--mount type=bind,source=/path/on/volume/to/host.access.log,target=/var/log/nginx/log/host.access.log                   \
--mount type=bind,source=/path/on/volume/to/letsencrypt/,target=/var/log/nginx/log/letsencrypt/                         \
-e SERVER_NAME=<domain-for-cert>                                                                                        \
-e CERT_EMAIL=<email-address-for-cert>                                                                                  \
-e SERVICE_NAME=<downstream-service-name>                                                                               \
-e SERVICE_PORT=80                                                                                                      \
-d                                                                                                                      \
-p 80:80                                                                                                                \
-p 443:443                                                                                                              \
webvariants/nginx-certbot
```
- Name this service
- Attach to the network we created earlier
- Mount an nginx configuration file from the persistent volume to the container
- Mount an nginx access log from the persistent volume to the container (mostly because the nginx-certbot container requires it)
- Mount the directory for SSL certificates from the persistent volume to the container
- Set up some environment variables for nginx and certbot
  - SERVER_NAME is the domain for the certificate
  - CERT_EMAIL is for letsencrypt
  - SERVICE_NAME is the name of whichever service traffic will be forwarded to
  - SERVICE_PORT is the port decent-slack-receiver is listening on
- Publish port 80 and 443 on the host to accept HTTP and HTTPS traffic
- Specify that we want the webvariants/nginx-certbot images

### 6. Create The RabbitMQ service

`docker service create -d --name=rabbitmq --network=decent-news-net rabbitmq:alpine`

### 7. Create The decent-slack-receiver Service
```
docker service create                         \
--name=decent-slack-receiver                  \
--network=decent-news-net                     \
--env-file decent-slack-receiver.env          \
-d                                            \
decentstudio/decent-slack-receiver:1.0-alpine
```

Note: The file `decent-slack-receiver.env` is stored on the persistent volume on digitalcoean.com.

### 8. Create The decent-news-distributor Service
```
docker service create                         \
--name=decent-news-distributor                \
--network=decent-news-net                     \
--env-file decent-news-distributor.env        \
-p 8081:80                                    \
-d                                            \
decentstudio/decent-news-distributor:1.0-alpine
```

Note: The file `decent-news-distributor` is stored on the persistent volume on digitalocean.com

### Conclusion

It took me several hours to get this deployment up. Most of that time was spent messing with Nginx and the HTTPS certificate. I finally slowed things down and made sure I knew what I was doing with each piece connected to Nginx and it worked out.

This deployment is functioning, but it needs improvements.

1. Multiple nodes are essential. The limitation right now is a lack of a network filesystem which multiple containers can mount for read/write.

2. The news distributor is not behind nginx at all right now. This is because I was exhausted after fighting with nginx-certbot for `in.decentstudio.com`. We will need another subdomain infront of the news distributor endpoint so that we can be sure to have proper load balancing and transparent access over port 80, rather than 8081 or some other variant.

3. Persistent storage for RabbitMQ is a must, currently it uses the disk on whatever host it is running on, but this goes along with a network file system solution in #1.

This is a fantastic first deploy. Each piece can be changed independently as we identify how this entire system should function. The most chaos will be between the distributor and the client as we refine the data format, but the internals of the receiver and distributor easily changed and deployed independently.

We do have a docker hub organization now. Our id is decentstudio. I'll be inviting everyone to that.

All of this manual setup is going to make it easier for us to implement CI/CD so that none of us have to mess with deployment after a certain point.

This document may not work exactly, I may have missed something along the way, but if anyone were to want to try this themselves I hope it will give most of the picture.
