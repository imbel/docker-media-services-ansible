## Docker Media Services

This project deploys a full home media server and monitoring suite onto a single host using Docker Swarm and Ansible. Ansible plays assume this is run from localhost.

### Services deployed are:

#### Reverse proxy and 2fa
    traefik
    authelia

#### Media services
    sabnzbd
    sonarr
    radarr
    jellyfin

#### Monitoring suite
    prometheus
    cadvisor
    node-exporter
    loki
    promtail
    grafana

### The defaults (following the steps) gets you:

* Traefik configured to be an internet facing reverse proxy with host rules for all services
* Automated certificate deployment and renewal through letsencrypt
* Redirection for each externally facing site to Authelia for 2fa
* A single user provisioned for authentication
* Docker logs, prometheus and host metrics available through Grafana, including some useful dashboards
* A suite of popular open source home media services focused on automation
    * Monitoring services setup is automated by this project, but the media services will need to be configured manually or by adding additional templating.

#### Note the following default access configurations

sabnzbd, sonarr, radarr, and grafana are all available externally, through traefik and authelia.

jellyfin is available through traefik with source IP filtering based on the cidr block defined in `internal_whitelist`.  This needs to be set as part of the below steps.

prometheus, cadvisor, node-exporter, loki and promtail have traefik disabled, but have port forwarding on the host.

### tl;dr

1. Install ansible per [official docs](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu)

```bash
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

1. Install ansible galaxy roles

```bash
ansible-galaxy install -p roles/ -r requirements.yaml 
ansible-galaxy collection install -r requirements.yaml 
```

1. I was having issues running `su - $USER` for each $USER that was added to the docker group, so their group membership is updated without a reboot.  Need to run this for each user manually for now

```bash
sudo su - $USER
```

1. This project only deploys one authelia user. Generate a hash for this users password to set in the next step.  See instructions [here](https://www.authelia.com/docs/configuration/authentication/file.html#passwords)

1. Set the following env vars populated with your secrets

```bash
export GRAFANA_ADMIN_PW=<secret> # This password will be set in grafana for the default user "admin"
export AUTHELIA_JWT_TOKEN=<secret> # https://www.authelia.com/docs/configuration/miscellaneous.html#jwt-secret
export AUTHELIA_SESSION_SECRET=<secret> # Secret used to encrypt session data
export SMTP_PW=<secret> # SMTP password for email notifications
export INIT_AUTH_USER_HASH=<secret> # Password hash previously generated
```

1. Most variables, like directories, have defaults that should work fine. There are however several values that need to be set before running anything.

* For all variables that are undefined, see comment inline for what's needed.
* Replace all instances of `example.com` with your external domain that services will be available on.
* Versions are all defaulted to latest. Tag specific versions, or don't.

var files to reference
```bash
ansible/group_vars/all.yaml
ansible/roles/media/vars/main.yaml
ansible/roles/traefik/vars/main.yaml
```

1. Deploy

```bash
ansible-playbook host_init.yaml
ansible-playbook deploy_services.yaml
```

1. Assuming all goes well, and that your server has ports 80/443 exposed to the internet, wait a bit and access your services at `<service>.<yourdomain>` ie; `https://sabnzbd.example.com`