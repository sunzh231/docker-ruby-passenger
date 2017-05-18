# Docker image with Ruby and Passenger, specialized for capistrano target

**NOITCE: I mainly used this in my own project, so use on your own risk, this might contained some opinionated settings.**

## Features
Based on [phusion/passenger-docker](https://github.com/phusion/passenger-docker), with following settings:

* Nginx enabled and Exposed 80 for nginx
* Passenger enabled
* Set to use `capistrano` as deployment tools
* `/home/app` as the root directory for `capistrano` deploy target
* Expose 22 for SSH access, so that capistrano can do the the deploy
* Following env variables will be passed to nginx and Rails app: `SECRET_KEY_BASE`,  `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`
* Set timezone to China timezone
* Use `zh_CN.UTF-8` as the locale
* `sidekiq` daemon will be launched if it has been integrated in this container, the configuration file should be located as `config/sidekiq.yml`

## Idea
The idea is pretty straight forward, create a container with `passenger` and `nginx` enabled, for running the Rails application. But, we will use `capistrano` for deploying, to support that, we need to allow `SSH` access to the container. So, the idea is to forward the `SSH` access to the container, so that the container can be used as a `capistrano` deployment target.

And, to make sure caches can be utilized, I used docker volume to storing the web application files, so that even recreating container won't take much time on subsequent `bundle install` and `assets:precompile`.

Why this way? Well, I tried, actually, I just love how `capistrano` works. I tried some other ways for using `docker` in Rails app, there are more or less issues, but mostly, what bothered me the most is, **slower** than `capistrano`.

## Usage

* Let's assume the container name will `foo` and the data vol will be named as `foo` as well.
* Assume the database is `postgres` and the container name is `foo-psql`
* Assume the redis server is `foo-redis`
* We will need SSH port to be forward, assume we will forward port `22` and `80` to `20022` and `20080`

```bash
# Create data vol
docker volume create --name foo
# Create the app container
docker run --name foo -d --restart="always" \
--link foo-psql:psql \
--link foo-redis:redis \
-e RAILS_ENV=production -e DB_HOST=psql -e DB_PORT=5432 \
-e DB_USER=postgres -e DB_PASSWORD=xxxxxx \
-e SECRET_KEY_BASE=xxxxxxxxxx \
-v foo:/home/app \
-p 0.0.0.0:20022:22 -p 0.0.0.0:20080:80 \
registry.cn-hangzhou.aliyuncs.com/pzgz/docker-ruby-passenger:ruby24-sidekiq

# Generate ssh key which will be used for checking out codes
docker exec foo ssh-keygen -f /root/.ssh/id_rsa -t rsa -N '' -C "foo-container"

# Get public key from the container, copy it and set it as the deploy key on git
docker exec foo cat /root/.ssh/id_rsa.pub

# Sometimes, you might need to copy authorized keys from host to container, so that you can easily login with SSH key
cat ~/.ssh/authorized_keys | docker exec -i foo bash -c "/bin/cat > /root/.ssh/authorized_keys"

# Then you can try the login with SSH key from your remote
ssh root@foo.bar.com -p 20022
```

## Known Issues

* Downloaded excel attachment generated by Axlsx appear to have weired filename(乱码) if it's in Chinese, root cause TBD
