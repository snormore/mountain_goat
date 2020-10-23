---
layout: post
title: Automate Rails server provisioning and deployment using Ansible and Capistrano
---

One of the hardest thing while learning Rails is to put your Rails application online on the internet so the public can access it. Many tutorials usually recommends Heroku as it is really simple to use, but the downside is that it can get expensive pretty fast, and their free plan has limitation that cold starts your dyno on inactivity and 10k rows limit for database (to force you to buy their paid plan, they need to pay server bills too).



Rails deployment can be hard if you have no prior sysadmin / devops experience, as you are setting up a server from scratch, and bound to came across issue like sudo permission, where to place files, what file to edit, cryptic error messages etc.



Chris Oliver has written an [excellent guide](https://gorails.com/deploy/ubuntu) for how to setup server and deploying Rails app, I have refered to this a lot when setting up server for my own side project and clients' app, and almost every time I setup a server following the guide, I would encounter at least one issue, either due to typo or some misconfigurations on my side.



Wouldn't it be great if there is a way to automate the Rails server provisioning? Like if I just run one command and it will setup everything for me? No more typo or misconfiguration issue and checking StackOverflow every time you want to deploy an app.



This has lead me into learning [Ansible,](https://github.com/ansible/ansible) which does the automation of the Rails server setup. This post will guide you on how to provision a server and deploy a Rails app using a preset Ansible script I wrote (it's the exact same as what you can achieve using GoRails' guide, but this time it is automated).



Table of contents :

1. Demo
2. Prerequisite
3. Install Ansible
4. Create your server with SSH key attached
5. Rails provisioning Ansible script
6. Preparing your Rails app for deployment
7. Deploy your Rails app with Capistrano
8. Updating environment variables
9. Debugging on Production
10. Further reading on Ansible



## Demo

Here's what you can achieve in this tutorial using Ansible, by typing the command `ansible-playbook server-provision.yml`  (server-provision.yml is the custom script I wrote), we can set up the server : 



<script id="asciicast-359424" src="https://asciinema.org/a/359424.js" async></script>

Server provisioned :
![provisioned site](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/site1.png)

After you make changes on the Rails app, commit or merge to master branch, then type `cap production deploy` to deploy the new changes, like how you click the "deploy" button in Heroku.

<script id="asciicast-wXe1n8uooDlAxJRZn7eVArFnk" src="https://asciinema.org/a/wXe1n8uooDlAxJRZn7eVArFnk.js" async></script>

![updated site](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/site2.png)



## Prerequisite

Before proceeding with this tutorial, make sure you
1. have generated SSH Key on your computer ([follow this guide if you haven't](https://docs.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent), make sure you have "id_rsa.pub" file located in `~/your_username/.ssh`)
2.  are using a Mac or Linux based OS
3. have a VPS hosting provider (I recommend using [DigitalOcean](https://m.do.co/c/f7f1b47b1fff), you can sign up [using this link](https://m.do.co/c/f7f1b47b1fff) to get $100 free credit!)



This tutorial will require you to create a Ubuntu 20.04 LTS server with your SSH key preconfigured later on.



## Install Ansible

Ansible is the automation tool we will be using to provision our server, let's install it.

**If you are using Mac**, you can install it with homebrew (easiest) :
`brew install ansible`



or if you don't have homebrew installed, you can install using pip :

check if pip is installed using `which pip` , if it is not installed :

`sudo easy_install pip`



then install Ansible using pip : 

`pip install ansible`



**If you are using Linux**, refer to your distro guide here : [https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu)



Once you have installed Ansible, move to the next step.



## Create your server with SSH key attached

I am using DigitalOcean as my VPS provider, you can use other VPS provider for this step as well.



Before creating a server, make sure you have your computer public key uploaded to the VPS provider. If you haven't, in DigitalOcean, go to **Settings** -> **Security** -> **SSH Key**, then select "**Add SSH Key**" : 

![settings](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/settings.png)



![add key](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/add_key.png)



In your terminal, type `nano ~/.ssh/id_rsa.pub` , then copy the content inside and paste into the form input and submit.



Next, proceed to create a droplet (server).

![create droplet](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/create_droplet.png)



Select '**Ubuntu 20.04**',  as the ansible script I wrote for this tutorial is designed for Ubuntu. You can try use other Ubuntu version but I have only tested it on Ubuntu 20.

![ubuntu 20](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/distro.png)

Select **at least 1GB RAM** for the server, as you will need quite some memory during Ruby compilation (compile Ruby source code to binary).

![ram requirement](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/ram.png)

Select **the SSH key of your computer**, so the server will be created with a root user that uses your public SSH key for authentication. This is important as the Ansible script will be authenticating as root user using your SSH public key (without using password), and the script will create a deploy user that uses the same SSH public key for authentication (that Capistrano will use to deploy the Rails app).



![ssh key](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/ssh_key.png)



Now proceed to create the droplet (server), and record down the IP address of the server.



## Rails provisioning Ansible script

Clone this repository (https://github.com/cupnoodle/rails-ansible), which contains the Ansible script I wrote to automate your rails server setup.

`git clone https://github.com/cupnoodle/rails-ansible`



then change directory to the repository :

`cd rails-ansible`



This repository contains two playbook file (think playbook like a sequence of automated task sent to your server). The **server-provision.yml** playbook will be responsible for :

1. Creating deploy user
2. Disabling password based login for strengthened security (can only login with public key authentication)
3. Installing Ruby, Nginx, Passenger, Postgresql, NodeJS and Redis
4. Setup a Sidekiq service file  (user-scoped), which you will be able to use later



We will use this to setup the server.



And another playbook **server-env.yml** is used to update the environment variables in your server, this is very handy when you have added new or updated existing environment variable and want the new value to reflect on the server. You can run this playbook each time you have updated the environment variable.



Before running the server provisioning script, there's a few variables I want you to change to suit your server needs.



Open up **vars/vars.yml**, you will see a few variables specified in the YAML file.

![Variables](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/variables.png)



Change the value of these variables to suit your needs.



**ruby_version**: the ruby version that will be installed on your server

**app_name**: your rails app name



**deploy_user**: the username of the user that will be created on the server, which will be used for deploying purpose

**deploy_password**: the password of the user



**deploy_user_public_key_local_path** : the path to the public key file in your computer (not the server), the script will copy the public key file specified in this path from your computer to the server, so next time when you login as the deploy user, it will authenticate using this public key, as the password based login will be disabled to strengthen security.  **No need to change this unless you have placed your public key file purposely in other location**, when you use ssh-keygen, your public key will by default located in "~/.ssh/id_rsa.pub".



**postgresql_db_user:** username of the PostgreSQL database user

**postgresql_db_password**: the password of the PostgreSQL database user

**postgresql_db_name**: the database name, usually you would use (app_name)_production for this



For the postgresql_version and locales, usually you won't need to change this unless you have specific need.



Next, open up **vars/env.yml**, these are your environment variables. Please note that you must change the RAILS_MASTER_KEY and SECRET_KEY_BASE value to match your Rails app value, else you will get error when running the app on the server.



![environments](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/environments.png)



**DATABASE_HOST**: the host of your database, as this ansible script will install the PostgreSQL database in the same server as the web server (and also app server), this should be **localhost** unless you are using RDS or some other remotely hosted database.



**DATABASE_NAME**: the database name, we are using the value of the {{ postgres_db_name }} variable which we have defined in the vars.yml file earlier, no need to change this unless you are using RDS or some other remotely hosted database.



**DATABASE_USERNAME**: the database username, we are using the value of the {{ postgres_db_name }} variable which we have defined in the vars.yml file earlier, no need to change this unless you are using RDS or some other remotely hosted database.



**DATABASE_PASSWORD**: the database username, we are using the value of the {{ postgres_db_name }} variable which we have defined in the vars.yml file earlier, no need to change this unless you are using RDS or some other remotely hosted database.



**RAILS_MASTER_KEY**: master key of your Rails app, please change this to the content of your Rails's app master.key file.



You can access the master.key file in **app/config/master.key**.

![master_key](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/master_key.png)





**SECRET_KEY_BASE**: the secret key used by your Rails app, please change this to your Rails app 's secret key base value. You can get this value by opening terminal, navigate to your rails app directory, and run `EDITOR=nano rails credentials:edit`



<script id="asciicast-erleGZjdmaaFSLUrWYUKE8833" src="https://asciinema.org/a/erleGZjdmaaFSLUrWYUKE8833.js" async></script>



You can add additional environment variables in this file (envs.yml) as required, eg: 

```yaml
---
environment_vars:
  DATABASE_HOST: localhost
  DATABASE_NAME: "{{ postgres_db_name }}"
  DATABASE_USERNAME: "{{ postgres_db_user }}"
  DATABASE_PASSWORD: "{{ postgres_db_password }}"

  RAILS_MASTER_KEY: aaaa
  SECRET_KEY_BASE: bbbbb
  
  SOME_KEY: value
  BIG_KEY: abcasdhjk
```



After making changes to the **vars.yml** and **env.yml** files (remember to save), open the **hosts.ini** file.



Change the IP address below the line **[web]** to your newly created server IP address : 

![ip address](https://rubyyagi.s3.amazonaws.com/4-automate-rails-provision-deploy-ansible/host_ip.png)



Alright now that we have finished the configuration, we can run the automated provisioning script now.



Navigate your terminal to the rails-ansible folder if you haven't already : 

`cd rails-ansible`



then run :

`ansible-playbook server-provision.yml`



This will run the server provisioning tasks, type "**yes**" and press enter if it ask "Are you sure you want to continue connecting".



This process will take around ~20 mins, as it will take 15mins for compiling Ruby from source code (using `rbenv install`).



After the provisioning task is complete, then run :

`ansible-playbook server-env.yml`



This will update the environment variable on the server.



At this point your server has completed setup, yay! 🥳 But if you visit your server's IP on web browser now, you will get a "404 not found" error message, this is because we haven't deploy our Rails app to the server yet, which we will do in the next section.



## Preparing your Rails app for deployment

For deployment, we will be using the [capistrano](https://github.com/capistrano/capistrano) gem and some add-ons. Capistrano will SSH to the server (using the deploy user we created earlier), git clone the repository, then run deployment task such as **rake db:migrate**, **rake assets:precompile** , restarting the server, etc.



Open your app's **Gemfile**, add these gems :

```ruby
gem 'capistrano', '~> 3.14'
gem 'capistrano-rails', '~> 1.6'
gem 'capistrano-rails-console', require: false
gem 'capistrano-passenger'
gem 'capistrano-rbenv'
```



Install these gems by running :

```ruby
bundle install
```



Then install Capistrano to your Rails app using :

```ruby
cap install STAGES=production
```



**production** is one of the stages. On big commercial project, usually there's multiple stages like QA, staging, production, each stage has different server and configuration, and you can choose which stage to deploy to, if you have multiple server / stage, you can install it `cap install STAGES=qa,staging,production` .



After installing capistrano, it will generates a few files :
This generates several files for us:



- `Capfile`
- `config/deploy.rb`
- `config/deploy/production.rb`



Edit the **Capfile** to look like this :

<script src="https://gist.github.com/cupnoodle/e931b023c56f795591693514b8606950.js"></script>



If you are using Sidekiq in your rails app, you can uncomment the line between **namespace :sidekiq do** and **after 'deploy:failed', 'sidekiq:restart'** , these line will stop your Sidekiq instance temporarily before deployment (job in queue will be stored and rerun after deployment) , and start your Sidekiq instance back after deployment to prevent interruption. 





Next, open **config/deploy.rb** , and modify it to look like this : 

<script src="https://gist.github.com/cupnoodle/b335361f00c34d893fb796600eea118b.js"></script>



And lastly, open **config/deploy/production.rb** , and change the IP address to your server IP address, and change the user used to deploy to the user variable we defined in deploy.rb (ie. the **set :user** line)

```ruby
# config/deploy/production.rb
server '1.2.3.4', user: fetch(:user), roles: %w{app db web}
```

<br>



Remember to save and commit the changes, and push to your remote Git repository.

```bash
git add -A
git commit -m "preparing Capistrano deployment"
git push origin master
```

<br>



Now we have finish preparing the Rails app for deployment, we are going to learn how to deploy in the next section.



## Deploy your Rails app with Capistrano

Navigate to your Rails app in the terminal if you haven't already, then run

`cap production deploy`



That's it! Once you run this command, your computer will SSH into the server, then the server will git clone the repository onto the server,  and run rake db:migrate, rake assets:precompile, restart passenger (and restart sidekiq if you have uncommented) etc, once it is done, you can go to your server URL and view the deployed Rails app! 🚀



Next time when you want to deploy new feature or changes, just commit your changes and push to remote, then run `cap production deploy` again!



## Updating environment variables

Let's say you have integrated Stripe payment recently and want to add the API key into environment variable. You can open up the ansible script repository (rails-ansible), modify the **vars/envs.yml** file to add your new Stripe API key : 

```yaml
---
environment_vars:
  DATABASE_HOST: localhost
  DATABASE_NAME: "{{ postgres_db_name }}"
  DATABASE_USERNAME: "{{ postgres_db_user }}"
  DATABASE_PASSWORD: "{{ postgres_db_password }}"

  RAILS_MASTER_KEY: aaaa
  SECRET_KEY_BASE: bbbbb
  
  SOME_KEY: value
  BIG_KEY: abcasdhjk
  STRIPE_SECRET_KEY: bruh
```

<br>



Then open Terminal, navigate to the ansible script folder 

`cd rails-ansible`



then run the server-env.yml playbook again :

`ansible-playbook server-provision.yml`



This will update the environment variables on your server, and restart the Nginx server so that the new environment variables will be reflected on your Rails app.



## Debugging on Production

If you stumble into an error page on your app, you can check the log at these locations in your server :

1. /home/deploy/myapp/current/log/production.log
2. /var/log/nginx/error.log



If you would like to do **rails:console** on your production server, you can do so by navigating to your Rails app folder in your local computer, then run `cap production rails:console`.



## Further reading on Ansible

You can reuse this ansible script for setting up another server, just change vars.yml, envs.yml and the hosts.ini files to suit your new server use.

 

Essentially Ansible automates your manual command like "sudo apt-get install" into a repeatable script. I recommend the book [Ansible for Devops](https://www.ansiblefordevops.com) by Jeff Geerling if you are interested on learning Ansible.


