---
layout: post
title:  "Деплой RoR через capistrano"
image: ''
date:   2017-03-01 17:19:35
tags:
- rails
- deploy
- AWS
- capistrano
- VPS
description: 'Деплой Rails приложений c помощью capistrano на примере сервера Amazon AWS EC2, Digital Ocean и других VPS или выделенных серверов'
categories:
- Ruby on Rails
- DevOps
---

## 1. Создаем инстанс Amazon AWS EC2 

В этом примере, я буду использовать **Ubuntu Server 16.04 LTS**.

Запускаем создание инстанса, кнопка **Launch Instance**

Выбираем ОС и различные параметры.

На 6 шаге в **Configure Security Group** можно сразу задать правила для доступа, например открываем наружу 80 или 443 порт, а доступ по SSH на 22 порт разрешаем только себе, заодно для себя можно открыть доступ к порту 2812, его использует веб-интерфейс сервиса <code>monit</code> который мы будем использовать для управления сервисами.

Жмем кнопку **Review and Launch** проверяем конфигурацию и жмем **Launch** появится вопрос о создании или успользовании существующей пары ключей, если ключа нет, создаем новый и скачиваем его.

Завершаем создание нажатием на кнопку **Launch Instance**

Для управления инстансами из своей консоли можно установить утилиту <a href="http://docs.aws.amazon.com/cli/latest/userguide/installing.html" target="_blank">AWS CLI</a>

Создаем БД RDS если требуется и настраиваем доступ через Security Group - Inbound Rules.

Для подключения к инстансу по SSH используем сохраненный ключ:

<code>ssh -i /path/my-key-pair.pem ec2-user@ec2-198-51-100-1.compute-1.amazonaws.com</code>

По умолчанию используется пользователь <code>ubuntu</code>, его и будем использовать для деплоя.

## 2. Настраиваем сервер


Настраиваем сервер с помощью скрипта или вручную:

{% highlight bash %}

#!/bin/bash

# INSTALLATION SCRIPT

uname -srm
cat /etc/*-release

USER_NAME=ubuntu
HOME_PATH="/home/$USER_NAME"

# VERSIONS

DEPLOYER_RVM_RUBY_VERSION=ruby-2.4.0
DEPLOYER_RVM_GEMSET_NAME=qna

# Задаем пароль пользователю ubuntu
# DEFINE PASSWORDS

DEPLOYER_PASSWORD=`openssl rand -base64 24`

echo "$USER_NAME:$DEPLOYER_PASSWORD" | chpasswd

# SET LANG VARS

echo 'export LC_ALL="en_US.UTF-8"'   >> ~/.bashrc
echo 'export LANGUAGE="en_US:en"'    >> ~/.bashrc
echo 'export LANG="en_US.UTF-8"'     >> ~/.bashrc
echo 'export LC_CTYPE="en_US.UTF-8"' >> ~/.bashrc

source ~/.bashrc

# VIM OPTIONS

echo "set nocompatible" > ~/.vimrc
echo ":set backspace=indent,eol,start" >> ~/.vimrc

echo "set nocompatible" >> $HOME_PATH/.vimrc
echo ":set backspace=indent,eol,start" >> $HOME_PATH/.vimrc
chown $USER_NAME:$USER_NAME $HOME_PATH/.vimrc
chmod 0644 $HOME_PATH/.vimrc

# FOR GEM INSTALL

echo 'gem: --no-ri --no-rdoc' >> $HOME_PATH/.gemrc
chown $USER_NAME:$USER_NAME $HOME_PATH/.gemrc
chmod -R 0644 $HOME_PATH/.gemrc

# CREATE SWAP

dd if=/dev/zero of=/swapfile bs=1M count=1024
chmod 0600 /swapfile
mkswap /swapfile
swapon /swapfile

echo "/swapfile none swap sw 0 0" >> /etc/fstab

echo 10 | tee /proc/sys/vm/swappiness
echo vm.swappiness = 10 | tee -a /etc/sysctl.conf
sysctl --system

# MINIMAL SOFT PACK

apt-get clean
apt-get update && apt-get upgrade

apt-get install -y coreutils htop monit unzip wget curl mc vim zsh git-core

sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

# REQUIREMENTS LIBS

apt install libmysqlclient-dev libpq-dev

# IMAGE MAGICK

apt-get install -y imagemagick libmagickwand-dev
convert --version

# REDIS

apt-get install -y redis-server
cp /etc/redis/redis.conf /etc/redis/redis.conf.default
systemctl restart redis

# NGINX

sudo apt-get install nginx -y

# SPHINX SEARCH

apt-get install sphinxsearch

# Устанавливаем PostgreSQL если нужно

apt-cache pkgnames postgresql
apt-get install -y postgresql postgresql-contrib libpq-dev

# RVM REQUIREMENTS

gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
\curl -sSL https://get.rvm.io | bash -s stable
/usr/local/rvm/bin/rvm requirements

(echo 'yes') | (/usr/local/rvm/bin/rvm implode)


echo "--------------------------------------------------" >> $HOME_PATH/credentials.txt
echo "DEPLOYER LOGIN/PASSWORD"                            >> $HOME_PATH/credentials.txt
echo "--------------------------------------------------" >> $HOME_PATH/credentials.txt

echo "
LOGIN: $USER_NAME
PASSWORD: $DEPLOYER_PASSWORD
"  >> $HOME_PATH/credentials.txt

echo "--------------------------------------------------" >> $HOME_PATH/credentials.txt
echo "SERVER IP ADDRESS PARAMS"                           >> $HOME_PATH/credentials.txt
echo "--------------------------------------------------" >> $HOME_PATH/credentials.txt

echo "" >> $HOME_PATH/credentials.txt

ifconfig eth0 | grep inet | awk "{print $2}" | sed "s/addr://" >> $HOME_PATH/credentials.txt

cat $HOME_PATH/credentials.txt

{% endhighlight %}

## 3. Настраиваем Capistrano

В <code>Gemfile</code> пишем:

{% highlight ruby %}
group :development do
  # Use Capistrano for deployment
  gem 'capistrano', require: false
  gem 'capistrano-bundler', require: false
  gem 'capistrano-rails', require: false
  gem 'capistrano-rvm', require: false
  gem 'capistrano3-puma',   require: false
  gem 'capistrano-sidekiq', require: false
end
{% endhighlight %}

Запускаем комманду <code>cap install</code>, это создаст файлы:

<code>config/deploy.rb</code>
<code>config/deploy/staging.rb</code>
<code>config/deploy/production.rb</code>
<code>Capfile</code>
И директорию <code>lib/capistrano/tasks</code>

В <code>Capfile</code> добавляем нужные инструменты:

{% highlight ruby %}

# Load DSL and set up stages
require 'capistrano/setup'

# Include default deployment tasks
require 'capistrano/setup'
require 'capistrano/deploy'

require 'capistrano/rails'
require 'capistrano/bundler'
require 'capistrano/rvm'
require 'capistrano/puma'
require 'capistrano/puma/nginx'

require 'capistrano/sidekiq'
require 'capistrano/sidekiq/monit' #to require monit tasks # Only for capistrano3

require 'whenever/capistrano'
require 'thinking_sphinx/capistrano'

install_plugin Capistrano::Puma  # Default puma tasks
# install_plugin Capistrano::Puma::Workers  # if you want to control the workers (in cluster mode)
# install_plugin Capistrano::Puma::Jungle # if you need the jungle tasks
install_plugin Capistrano::Puma::Monit  # if you need the monit tasks
install_plugin Capistrano::Puma::Nginx  # if you want to upload a nginx site template

{% endhighlight %}

Список доступных задач можно посмотреть в консоли  <code>cap -T</code>

Настраиваем конфигурацию в <code>config/deploy.rb</code>:

{% highlight ruby %}

# config valid only for current version of Capistrano
lock "3.8.0"

set :repo_url,        'git@github.com:EugeneKey/QnA.git'
set :application,     'qna'
set :user,            'ubuntu'
set :puma_threads,    [4, 16]
set :puma_workers,    0

# Default branch is :master
# ask :branch, `git rev-parse --abbrev-ref HEAD`.chomp

# Default value for :linked_files is []
set :linked_files, fetch(:linked_files, [])
        .push('config/database.yml', 'config/private_pub.yml', 
              'config/private_pub_puma.rb', '.env')

# Default value for linked_dirs is []
set :linked_dirs, fetch(:linked_dirs, [])
        .push('bin', 'log', 'tmp/pids', 'tmp/cache', 
              'tmp/sockets', 'vendor/bundle', 'public/system', 
              'public/uploads')

namespace :puma do
  desc 'Create Directories for Puma Pids and Socket'
  task :make_dirs do
    on roles(:app) do
      execute "mkdir #{shared_path}/tmp/sockets -p"
      execute "mkdir #{shared_path}/tmp/pids -p"
    end
  end

  before :start, :make_dirs
end

namespace :private_pub do
  desc 'Start private_pub server'
  task :start do
    on roles(:app) do
      within current_path do
        with rails_env: fetch(:rails_env) do
          execute :bundle, "exec pumactl -F config/private_pub_puma.rb start"
        end
      end
    end
  end

  desc 'Stop private_pub server'
  task :stop do
    on roles(:app) do
      within current_path do
        with rails_env: fetch(:rails_env) do
          execute :bundle, "exec pumactl -F config/private_pub_puma.rb stop"
        end
      end
    end
  end

  desc 'Restart private_pub server'
  task :restart do
    on roles(:app) do
      within current_path do
        with rails_env: fetch(:rails_env) do
          execute :bundle, "exec pumactl -F config/private_pub_puma.rb restart"
        end
      end
    end
  end
end

namespace :deploy do
  desc "Make sure local git is in sync with remote."
  task :check_revision do
    on roles(:app) do
      unless `git rev-parse HEAD` == `git rev-parse origin/master`
        puts "WARNING: HEAD is not the same as origin/master"
        puts "Run `git push` to sync changes."
        exit
      end
    end
  end

  desc 'Initial Deploy'
  task :initial do
    on roles(:app) do
      before 'deploy:restart', 'puma:start', 'private_pub:start'
      invoke 'deploy'
    end
  end

  desc 'Restart application'
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      invoke 'puma:restart'
      invoke 'private_pub:restart'
    end
  end

  before :starting,     :check_revision
  after  :finishing,    :compile_assets
  after  :finishing,    :cleanup
  after  :finishing,    :restart

end

{% endhighlight %}

В файле <code>config/deploy/production.rb</code> указываем сервер, путь к rsa ключу для подключения и остальные параметры:

{% highlight ruby %}

server '123.123.123.123', port: 22, roles: [:web, :app, :db], primary: true
set :ssh_options,     { forward_agent: true, user: fetch(:user), 
                        keys: %w(~/.ssh/amazon_rsa_key.pem), 
                        auth_methods: %w(publickey password) }
# server 'example.com', user: 'deploy', roles: %w{app web}, other_property: :other_value
# server 'db.example.com', user: 'deploy', roles: %w{db}

set :rails_env, :production

set :pty,             false
set :use_sudo,        false
set :stage,           :production
set :deploy_via,      :remote_cache
set :deploy_to,       "/home/#{fetch(:user)}/#{fetch(:application)}"
set :puma_bind,       "unix://#{shared_path}/tmp/sockets/#{fetch(:application)}-puma.sock"
set :puma_state,      "#{shared_path}/tmp/pids/puma.state"
set :puma_pid,        "#{shared_path}/tmp/pids/puma.pid"
set :puma_access_log, "#{release_path}/log/puma.error.log"
set :puma_error_log,  "#{release_path}/log/puma.access.log"
set :puma_preload_app, true
set :puma_worker_timeout, nil
set :puma_init_active_record, true  # Change to false when not using ActiveRecord

set :nginx_server_name, 'myrailsproject.com'

set :sidekiq_options_per_process, ["--queue default --queue mailers"]

{% endhighlight %}

После настройки параметров можно пробовать деплоить приложение.

Для этого, коммитим изменения на github:
{% highlight bash %}
git add .
git commit -am "Name commit"
git push
{% endhighlight %}
И выполняем
{% highlight ruby %}
cap production deploy:initial
{% endhighlight %}

Если появляются ошибки исправляем, коммитим изменения и пробуем заново.

## Настраиваем конфигурацию monit

После успешного деплоя, для тконтроля и управления сервисами можно использовать <code>monit</code>.

Создаем конфиг <code>monit</code>, добавляем нужные сервисы:

{% highlight bash %}
### Unicorn ###
check process unicorn
  with pidfile "/home/deployer/qna/current/tmp/pids/unicorn.pid"
  start program = "/bin/su - deployer -c 'cd /home/deployer/qna/current && ( RAILS_ENV=production ~/.rvm/bin/rvm default do bundle exec unicorn -c /home/deployer/qna/current/config/unicorn/production.rb -E deployment -D  )'"
  stop program = "/bin/su - deployer -c 'cd /home/deployer/qna/current && /usr/bin/env kill -s QUIT `cat /home/deployer/qna/current/tmp/pids/unicorn.pid`'"
  if memory usage > 90% for 3 cycles then restart
  if cpu > 90% for 2 cycles then restart
  if 5 restarts within 5 cycles then timeout

### Puma ###
check process puma
  with pidfile "/home/ubuntu/qna/shared/tmp/pids/puma.pid"
  start program = "/bin/su - ubuntu -c 'cd /home/ubuntu/qna/current && ~/.rvm/bin/rvm default do bundle exec puma -C /home/ubuntu/qna/shared/puma.rb --daemon'"
  stop program = "/bin/su - ubuntu -c 'cd /home/ubuntu/qna/current && ~/.rvm/bin/rvm default do bundle exec pumactl -S /home/ubuntu/qna/shared/tmp/pids/puma.state stop'"
  if cpu > 80% then restart
  if memory usage > 80% for 2 cycles then restart
  if 3 restarts within 3 cycles then timeout

### Nginx Passenger ###
check process nginx
  with pidfile /opt/nginx/logs/nginx.pid
  start program = "/etc/init.d/nginx start"
  stop program = "/etc/init.d/nginx stop"
  if cpu > 60% for 2 cycles then alert
  if cpu > 80% for 5 cycles then restart
  if memory usage > 80% for 5 cycles then restart
  if failed host 127.0.0.1 port 80 protocol http then restart
  if 3 restarts within 5 cycles then timeout

### Nginx ###
check process nginx with pidfile /var/run/nginx.pid
  group www
  group nginx
  start program = "/etc/init.d/nginx start"
  stop program = "/etc/init.d/nginx stop"
  if cpu > 60% for 2 cycles then alert
  if cpu > 80% for 5 cycles then restart
  if memory usage > 80% for 5 cycles then restart
  if failed port 80 protocol http request "/" then restart
  if 5 restarts with 5 cycles then timeout
  depend nginx_bin
  depend nginx_rc

check file nginx_bin with path /usr/sbin/nginx
  group nginx
  include /etc/monit/templates/rootbin

check file nginx_rc with path /etc/init.d/nginx
  group nginx
  include /etc/monit/templates/rootbin

### Postgresql ###
check process postgresql
  with pidfile "/var/run/postgresql/9.3-main.pid"
  start program = "/usr/sbin/service postgresql start"
  stop  program = "/usr/sbin/service postgresql stop"
  if failed host localhost port 5432 protocol pgsql then restart
  if cpu > 80% then restart
  if memory usage > 80% for 2 cycles then restart
  if 5 restarts within 5 cycles then timeout

### Redis ###
check process redis-server
    with pidfile "/var/run/redis/redis-server.pid"
    start program = "/etc/init.d/redis-server start"
    stop program = "/etc/init.d/redis-server stop"
    if totalmem > 100 Mb then alert
    if children > 255 for 5 cycles then stop
    if cpu usage > 95% for 3 cycles then restart
    if memory usage > 80% for 5 cycles then restart
    if failed host 127.0.0.1 port 6379 then restart
    if 5 restarts within 5 cycles then timeout

### Sidekiq ###
check process sidekiq
  with pidfile "/home/ubuntu/qna/shared/tmp/pids/sidekiq-0.pid"
  start program = "/bin/su - ubuntu -c 'cd /home/ubuntu/qna/current && ~/.rvm/bin/rvm default do bundle exec sidekiq  --index 0 --pidfile /home/ubuntu/qna/shared/tmp/pids/sidekiq-0.pid --environment production  --logfile /home/ubuntu/qna/shared/log/sidekiq.log   --queue default --queue mailers -d'" with timeout 30 seconds
  stop program = "/bin/su - ubuntu -c 'cd /home/ubuntu/qna/current && ~/.rvm/bin/rvm default do bundle exec sidekiqctl stop /home/ubuntu/qna/shared/tmp/pids/sidekiq-0.pid'" with timeout 20 seconds
  if cpu > 80% then restart
  if memory usage > 80% for 2 cycles then restart
  if 3 restarts within 3 cycles then timeout

### Sphinx ###
check process sphinx
  with pidfile "/home/deployer/qna/shared/log/production.sphinx.pid"
  start program = "/bin/su - deployer -c 'cd /home/deployer/qna/current && /home/deployer/.rvm/bin/rvm default do bundle exec rake RAILS_ENV=production ts:start'"
  stop program = "/bin/su - deployer -c 'cd /home/deployer/qna/current && /home/deployer/.rvm/bin/rvm default do bundle exec rake RAILS_ENV=production ts:stop'"
  if cpu > 80% then restart
  if memory usage > 80% for 2 cycles then restart
  if 3 restarts within 3 cycles then timeout

### Thin (private_pub) ###
check process thin
  with pidfile "/home/deployer/qna/shared/tmp/pids/thin.pid"
  start program = "/bin/su - deployer -c 'cd /home/deployer/qna/current && RAILS_ENV=production /home/deployer/.rvm/bin/rvm default do bundle exec thin -C config/private_pub_thin.yml start'"
  stop program = "/bin/su - deployer -c 'cd /home/deployer/qna/current && RAILS_ENV=production /home/deployer/.rvm/bin/rvm default do bundle exec thin -C config/private_pub_thin.yml stop'"
  if cpu > 80% then restart
  if memory usage > 80% for 2 cycles then restart
  if 3 restarts within 3 cycles then timeout

### Puma (private_pub) ###
check process private_pub
  with pidfile "/home/ubuntu/qna/shared/tmp/pids/puma_private_pub.pid"
  start program = "/bin/su - ubuntu -c 'cd /home/ubuntu/qna/current && RAILS_ENV=production ~/.rvm/bin/rvm default do bundle exec pumactl -F /home/ubuntu/qna/shared/config/private_pub_puma.rb start'"
  stop program = "/bin/su - ubuntu -c 'cd /home/ubuntu/qna/current && RAILS_ENV=production ~/.rvm/bin/rvm default do bundle exec pumactl -F /home/ubuntu/qna/shared/config/private_pub_puma.rb stop'"
  if cpu > 80% then restart
  if memory usage > 80% for 2 cycles then restart
  if 3 restarts within 3 cycles then timeout

{% endhighlight %}

перезапускаем monit:

<code>sudo monit reload</code>

по умолчанию веб-интерфейс отключен, включем в конфиге <code>/etc/monit/monitrc</code>, раскоментировав нужные строки:

{% highlight bash %}
set httpd port 2812 and                                                     
#     use address localhost  # only accept connection from localhost        
#     allow localhost        # allow localhost to connect to the server and 
    allow 10.0.2.2                                                          
    allow admin:monit      # require user 'admin' with password 'monit'
{% endhighlight %}

## Особенности деплоя на другие VPS серверы

У Digital Ocean и других VPS, процесс ничем не отличается, но скорее всего потребуется создать пользователя для деплоя, так как изначально дается рутовый доступ по SSH, а это не очень хорошо.

После создания пользователя нужно создать RSA ключ и настроить доступ по SSH для этого пользователя с использованием ключа и отключить доступ для root пользователя.

Есть еще популярный вариант, это использовать при разработке и деплое <code>Docker</code>, об этом напишу в другой раз.


