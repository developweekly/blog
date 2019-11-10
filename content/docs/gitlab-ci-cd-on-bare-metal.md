---
title: "Gitlab CI/CD On Bare Metal"
date: 2019-11-10T13:26:29+03:00
draft: false
---

# Gitlab CI/CD On Bare Metal

**Nov 10, 2019**
<!-- <sup>Last modified: **Dec 2, 2018**</sup> -->

I'll skip the part why you should embrace the devops culture and start migrating your legacy code to containers and everything. This post should help you if you got to the point of finding out how it works if you were to operate a self-hosted CI/CD system.

We adopted Gitlab as our CI/CD server and starting automating everything, from configuration to deployment. Automating your workflow helps developers skip spending time on toil and instead focus on delivering better software.

This setup runs on bare metal, on Ubuntu Servers with a multi-manager Docker Swarm cluster running on them. Managing such a setup requires somewhat deep understanding how containers work.

Every single piece of software runs in a container as a docker service and deployed within Gitlab CI/CD pipelines, even Gitlab Server itself. This is a nice abstraction that helps a lot when managing hundreds of services.

We use HAProxy as a reverse proxy, it handles all routing of traffic to and from those underlying services.

# Configuring HAProxy for Gitlab Server

Before we install Gitlab, let's configure the ports it will be served. I will cover only relevant parts of the configuration.

To reserve ssh port on the host machine, we will use a different TCP port for SSH communications on Gitlab server.

```haproxy
frontend gitlab_ssh
    bind *:2289
    mode tcp
    option tcplog
    tcp-request connection accept if { src 10.255.0.0/16 172.17.0.0/16 192.168.7.0/24 }
    default_backend gitlab_ssh_backend
```

Here all the tcp connections coming to HAProxy container from port `2289` are redirected to `gitlab_ssh_backend`, with some restriction to local subnets.

```haproxy
backend gitlab_ssh_backend
    mode tcp
    server gitlab_ssh_backend_service service.algosis.com.tr:2290 check
```

And this is the backend configuration, to identify which `host:port` Gitlab server is listening TCP connections. Here it is `service.algosis.com.tr:2290`.

We need another configuration for the web port. We want all http traffic to redirect to https port first. This redirect config is also used for every http connections coming from port `80`.

```haproxy
frontend www-http
    bind *:80
    http-request set-header X-Forwarded-Proto http
    reqadd X-Forwarded-Proto:\ http
    redirect scheme https if !{ ssl_fc }
```

Now here is the relevant https configuration. We'll run Gitlab server under `gitlab.algosis.com.tr`.

```haproxy
frontend www-https
    bind *:443 ssl crt /etc/crt/algosis.com.tr.pem crt

    # Gitlab web server
    acl gitlab_algosis hdr_sub(host) -i gitlab.algosis.com.tr
    http-request set-log-level silent if gitlab_algosis
    use_backend gitlab_web_backend if gitlab_algosis

    # Gitlab Container Registry
    acl localip src 10.255.0.0/16 172.17.0.0/16 192.168.7.0/24
    acl registry hdr_sub(host) -i registry.algosis.com.tr
    use_backend registry_backend if registry localip

backend gitlab_web_backend
    mode http
    option forwardfor
    server gitlab_web_backend_service service.algosis.com.tr:8089 check

backend registry_backend
    mode http
    option forwardfor
    server registry_backend_service service.algosis.com.tr:12557 check
```

And the web backend of Gitlab server listens on `service.algosis.com.tr:8089`.

Note that we also handle certificates on HAProxy, and won't be providing Gitlab server a different certificate.

Gitlab is going to explode your HAProxy logs. If you want to filter them, you can add `set-log-level silent` on your https-frontend.

We also use Gitlab Container registry for storing and serving all of our container images. It runs on a different domain `registry.algosis.com.tr` and on port `12557`. We also restricted its access on local subnet.

Now we can configure Gitlab server to listen TCP and HTTPS connections on those ports.

# Deploying Gitlab Server

Gitlab is deployed as a container, just as rest of the services. We will use Gitlab's official Docker image. Since the container will be run as a Docker service, we will create a docker-compose file and deploy it with `docker stack` command. We will do this manually right now since we don't have an automated CI/CD pipeline yet.

```yaml
version: '3.5'
services:
  web:
    image: 'gitlab/gitlab-ce:latest'
    hostname: 'gitlab.algosis.com.tr'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.algosis.com.tr'
        registry_external_url 'https://registry.algosis.com.tr'
        # Custom SSH port
        gitlab_rails['gitlab_shell_ssh_port'] = 2289
        # Backup configuration
        gitlab_rails['backup_upload_connection'] = {
          'provider' => 'AWS',
          'region' => 'ams3',
          'aws_access_key_id' => '<some-secret-value>',
          'aws_secret_access_key' => '<some-secret-value>',
          'endpoint'              => 'https://<some-secret-value>.digitaloceanspaces.com'
        }
        gitlab_rails['backup_upload_remote_directory'] = 'algosis'
        gitlab_rails['backup_keep_time'] = 604800
        # Gitlab schedules check interval
        gitlab_rails['pipeline_schedule_worker_cron'] = "* * * * *"
        # Gitlab Email Setup
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "smtp.yandex.com.tr"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "<some-secret-value>@algosis.com.tr"
        gitlab_rails['smtp_password'] = "<some-secret-value>"
        gitlab_rails['smtp_domain'] = "algosis.com.tr"
        gitlab_rails['gitlab_email_from'] = '<some-secret-value>@algosis.com.tr'
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_openssl_verify_mode'] = 'peer'
        # Web and Registry Port configs
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        registry_nginx['listen_port'] = 12557
        registry_nginx['listen_https'] = false
    ports:
      - '8089:80'
      - '2290:22'
      - '12557:12557'
    volumes:
      - gitlab_config:/etc/gitlab
      - gitlab_logs:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab
      - gitlab_ssh:/etc/ssh
      - ssl_certs:/certs
    deploy:
      restart_policy:
        condition: any
        delay: 5s
      placement:
        constraints:
          - node.hostname == service

volumes:
  gitlab_config:
    external: true
  gitlab_logs:
    external: true
  gitlab_data:
    external: true
  gitlab_ssh:
    external: true
  ssl_certs:
    external: true
```

Now this is a lot to take in, so we better cover it peace by peace.

I think image, hostname and url parts are clear from haproxy configurations. Since we used a custom ssh port on HAProxy to reserve default ssh port on the host, we define it in `gitlab_rails['gitlab_shell_ssh_port'] = 2289`.

The next part is backup configuration. Gitlab is capable of backing itself on an S3 bucket, so we configure our remote S3 bucket access keys. `gitlab_rails['backup_keep_time'] = 604800` means we want Gitlab to keep its backup files on local as long as 604800 seconds, which is 7 days.

We are going to use Gitlab Schedules to operate our scheduled pipelines. `gitlab_rails['pipeline_schedule_worker_cron'] = "* * * * *"` configuration tells Gitlab to check if there's a new schedule every second, default is 12 hours as far as I remember and it's pretty annoying.

You can configure a SMTP server if you want Gitlab to notify you on some certain operations such as pipeline failure, etc. This is really helpful when integrated with Mattermost - Gitlab's open source Slack alternative - with some fancy bots.

Web port and registry port is configured as aligned with those in HAProxy. Here the container port `8089` is linked to `80` so `nginx['listen_port'] = 80` and since HAProxy handles ssl certificates, `nginx['listen_https'] = false`. Don't worry, Gitlab is still configured to operate on an https domain. Similar configuration is applied for registry as well.

We want to persist some folders such as logs, git data and configurations, so we define docker volumes (externally) before we deploy this as a service. Also some docker service placement restrictions, but these are optional for now.

Save the file as `docker-stack.yml` and deploy it with `docker stack deploy --compose-file docker-stack.yml gitlab` and you should be good to go. It would take a while since docker needs to pull the image from Docker Hub registry before deploying the service.

Remember that for backup to run, we still need to trigger backup command with a scheduled pipeline, I will show it after we can use pipelines. For pipelines, we need what Gitlab calls a runner.

# Deploying Gitlab Runners

Runners are Gitlab's workers for your pipelines. Pipelines are configured with `.gitlab-ci.yml` on each project and runners run the code defined in those configuration. I highly suggest you to read Gitlab docs for [configuring Gitlab Runners](https://docs.gitlab.com/ee/ci/runners/).

For our setup, runners will also run as a container with a docker socket priviledge. This allows runners to create new containers on the underlying docker host. We'll use official Gitlab runner docker image.

Runners can use different executors such as bash, docker etc. In our cluster every piece of code runs through docker so we will use docker executor. To be able to run docker commands in a container, we will also bind the docker socket to it. And finally, we will deploy a runner container on each nodes of our docker cluster, which means the service will be deployed in `global` mode.

```yaml
version: '3.5'
services:
    runner:
      image: 'gitlab/gitlab-runner:latest'
      volumes:
        - '/var/run/docker.sock:/var/run/docker.sock'
        - gitlab-runner:/etc/gitlab-runner
      environment:
        - RUNNER_EXECUTER=docker
      deploy:
        mode: global
        restart_policy:
          condition: any
          delay: 5s

volumes:
  gitlab-runner:
    external: true
```

You see that every docker volume we create is external so we need to manually create required volumes before deploying this configurations. Save this configuration in a file and deploy it like we did in Gitlab section. `docker stack deploy --compose-file docker-stack.yml gitlab-runner`.

Now we have runner containers on each node but Gitlab and its runners are unaware of each other. Let's help them communicate.

# Registering Runners to Gitlab

Okay we now have bunch of Gitlab services running on our cluster. Next step is registering runner services to Gitlab so that it can start using them as pipeline workers. Again I strongly suggest you to read the official documentation for [registering runners](https://docs.gitlab.com/runner/register/#docker).

We will invoke `register` command in runner containers, with some configuration. This command is actually interactive and very intuitive. It asks you a couple of questions to help you configure your runner. But we want everything to be automated and run through the pipeline so we might as well figure out a non-interactive way to do so. We will use same runners across all projects, so our runners will be `shared` runners. We also deployed runners on each node of our clusted, so we better `tag` them appropriately to identify the host runner is working on. We will use `docker` executor, and we need a base docker image for the containers runners will create during executing pipeline jobs. Wow, containers everywhere, right?

Remember we wanted to be able to run docker commands in runners? That's why we will use official docker image for running docker in docker. In fact, it's called docker in docker (dind), and the image lies in [docker hub](https://hub.docker.com/_/docker). This docker image does not contain a lot of commands but for now it's sufficient. We'll later extend this image to add some capabilities such as running `docker-compose`.

I suggest you to go ahead and try the interactive register command in runners, but you might as well sneak a peak to my non-interactive command here.

```shell
docker exec -it $RUNNER_CONTAINER gitlab-runner register \
--non-interactive \
--docker-tlsverify \
--url https://gitlab.algosis.com.tr \
--tag-list service,docker-builder,docker-stack \
--registration-token <some-secret-value> \
--docker-image docker:latest \
--executor docker \
--docker-pull-policy if-not-present \
--run-untagged=true \
--locked=false \
--docker-volumes /var/run/docker.sock:/var/run/docker.sock
```

You can obtain your own registration token on Gitlab Admin panel under Overview/Runners section.

As you add more projects, you are going to need multiple runners on your docker swarm nodes. You can register new runners with the same command we used.

You should be able to see your runners on Gitlab admin panel with unique runner tokens, shared type, and tags you provided.

Now let's get those runners working.

# Your First .gitlab-ci.yml

For a detailed configuration documentation, read [official docs](https://docs.gitlab.com/ee/ci/yaml/README.html).

We deployed Gitlab manually, but we can let runners do the job for us. Keep the same `docker-stack.yml` we used for Gitlab, but add a `.gitlab-ci.yml` file as follows.

```yaml
image: docker:latest

stages:
  - deploy
  - update
  - backup

production:
  stage: deploy
  tags:
    - docker-stack
  script:
    - docker stack deploy --compose-file docker-stack.yml gitlab
  environment:
    name: production
  only:
   changes:
     - docker-stack.yml
  except:
    - schedules
```

Create a new project called `gitlab` on your Gitlab instance. After committing and pushing configuration files to your project, a pipeline job should start running and when succeeded, your Gitlab container should be re-created, by Gitlab runners.

# Adding Self-Update and Backup Capabilities to Gitlab

This pipeline won't be running again unless there's a commit to the project repository. But Gitlab releases are pretty frequent and you should keep it up to date. For this scheduled task, we can use Gitlab Schedules.

First, go to `gitlab` project page and create a new schedule under CI/CD / Schedules. You can use one of the pre-defined interval patterns, such as every day (at 4:00am).

Now we can add an update job to our `gitlab-ci.yml` configuration file.

```yaml
update:
  stage: update
  tags:
    - docker-stack
  script:
    - docker stack deploy --compose-file docker-stack.yml gitlab
  environment:
    name: production
  only:
   - schedules
```

Remember that we defined our own `stages` in configuration such as `deploy, update, backup`. Jobs will run in this order according to their stage, and jobs in the same stage will run parallel. We also don't want update job to be run on each commit, so we restrict it to be run only on `schedules`. Now you can test your schedule by playing play button on CI/CD / Schedules page.

Similarly, we can add a backup job as well.

```yaml
backup:
  stage: backup
  tags:
    - service
  script:
    - docker ps --filter health=healthy --filter name=gitlab_web --format "{{.ID}}" | xargs -t -I {} docker exec -t {} gitlab-backup create SKIP=registry,artifacts STRATEGY=copy DIRECTORY=gitlab_backups
  only:
   - schedules
```

We are filtering `gitlab_web` container and running `gitlab-backup` command on it. We also skip `registry,artifacts` since they would take a lot of space on backup. Read more on [official docs](https://docs.gitlab.com/ee/raketasks/backup_restore.html#backup-options).