# Custom source-to-image builder for static nginx containers

## Overview
This repo is source code to `docker build` an S2I build image for an Nginx server 
on CentOS7.  

Most people will just need the Basic Usage if you just want NGinx to serve 
HTML/CSS/JS/whatever from it and/or use it as a reverse or forward proxy.

For people who want to change the builder image see Advanced Usage.  Building a 
new S2I is for people who want to do more than simply change NGinx config and
serve static files.

## Basic Usage

### Step 1: Building the S2I Image

Long term, we should have this s2i image available in the global registry 
but until then, you'll have to build this custom s2i image.  Here's how:

Required tools install:

1. [Docker Toolbox](https://www.docker.com/products/docker-toolbox)

Open Docker QuickStart Terminal (need Docker engine started and env variables set) and
build the S2I image:

`$ docker build -t s2i-nginx-npm git://github.com/BCDevOps/s2i-nginx-npm`

Tag and push this image to the OpenShift Docker Registry for your OpenShift Project:

`$ docker tag s2i-nginx-npm docker-registry.pathfinder.gov.bc.ca/<yourprojectname>/s2i-nginx-npm`

`$ docker login docker-registry.pathfinder.gov.bc.ca -u <username> -p <token>`

`$ docker push docker-registry.pathfinder.gov.bc.ca/<yourprojectname>/s2i-nginx-npm`

[Forgot how to get a token?](https://console.pathfinder.gov.bc.ca:8443/console/command-line)

### Step 2: Customizing your Nginx Config and Static Files

Create a separate Git repo or [fork/clone our example](https://github.com/BCDevOps/s2i-nginx-npm-example) repo.

The directory structure should look like this:

`/dist/    <- where your npm run build should output to`

`/html/    <- your static files here, and the output of the npm build process`

`/conf.d/  <- your custom NGinx config files`

`/aux/     <- your custom auxiliary files`

All `*.conf` files in the `/conf.d/` will be copied and read by NGinx.  

All `*.conf.tmpl` files in the `/conf.d/` will be copied and `envsubst` will run replacing $ENVVAR with the run-time environment variable.

You can put auxiliary files in a directory `aux` (or `NGINX_AUX_DIR`) to copy
them to the resulting image, e.g. htpasswd files.  These will be copied to `/opt/app-root/etc/aux`.

Optionally can supply a `/nginx.conf-snippet` that will be used by the as built container.

### Steb 3a: Building the s2i-nginx-npm in OpenShift for Multiple Projects (Recommended)
Note: This is for apps where the build project is seperated from the deployment environments.

`$ oc process -f https://raw.githubusercontent.com/BCDevOps/s2i-nginx-npm/master/openshift/templates/rproxy-build-template.json -v BUILDER_IMAGESTREAM_TAG=s2i-nginx-npm:latest -v SOURCE_REPOSITORY_URL=<url to repo created in step 2> -v NGINX_PROXY_URL=<url to proxy to consistent across all envs> -v NAME=rproxy | oc create -f -`

Once build you can use another template to deploy the application to one of your project environments.  
A sample is provided in this example but feel free to customize your own. 

`$ oc process -f https://raw.githubusercontent.com/bcgov/mygovbc-nginx/master/openshift/templates/rproxy-environment-template.json -v APPLICATION_DOMAIN=<URL of the route you want to the rproxy> -v APP_DEPLOYMENT_TAG=dev | oc create -f - `

### Step 3b: Building the s2i-nginx-npm in OpenShift for Single Project
Note: This is for apps where the build and deployments are all-in-one.

The builder image should've built and been pushed to OpenShift.  You should have your own custom nginx conf and/or static files.  Now you can spin it up in your OpenShift Project.

1. [Login to OpenShift](https://console.pathfinder.gov.bc.ca:8443/console/)
2. Choose the project where you want this
3. Click `Add to Project`
4. Click `Don't see the image you are looking for?`
5. Scroll to find your `s2i-nginx-npm` S2I image and click it
6. Give it a name, e.g., `rproxy`, and the Git URL to the repo you made in Step 2
7. The defaults are generally fine for basic usage
8. Watch it deploy automatically!

## Best Practices

1. Generally you want a minimum of 2 pods for high availability purposes, e.g., rolling deployments.
2. For Single Project scenarios, branch, e.g, `master and development` your configuration repo to avoid deploying untested config to production.  
3. If reverse proxying to a OpenShift Service, remove the `route' from the service to avoid proxy by-pass.  This is especially important if the reverse proxy is providing a layer of security.
4. Harden your nginx configuration.   
5. Avoid environment specific configuration in your custom configuration repo, should be generic nginx config for ALL environments.

## Integrating with BC Government SiteMinder Web SSO

Integrating SiteMinder Web SSO, BC's current SSO standard, is fairly simple with this nginx as a proxy.  You'll need to on-board with IDIM to make this happen.  And some consulting charges apply.  

Tripling proxing and double load balancing adds some unnecessary latency and additional points of failure.  Not the ideal situation, but is the most feasible option at this time for the OpenShift environment, see Future and Alternate Considerations.  

### Network Flow  

This network flow describes how HTTP traffic is handle between the browser and eventually
your application on OpenShift.

1. Web Browser initiates HTTPS (certificate purchase required) request to a FQDN 
2. The FQDN resolves to WAM's SiteMinder reverse proxy shared service.  Note: it does path thru an load balancer but its  transparent at this HTTP level)
	1. WAM sets up this during your IDIM onboarding process
3. WAM's reverse proxy establishes a new TLS connection to a https://*.pathfinder.gov.bc.ca OpenShift router endpoint.  Note: it does path thru a load balancer but its  transparent at this HTTP level)
	1. You setup this new route to your new Nginx in your OpenShift project
	2. This is an TLS edge terminated route
4. OpenShift router establishes a new HTTP connection to the OpenShift Ngnix service. OpenShift router ALWAYS appends the client IP in the the X-Forwarded-For HTTP Header.
5. Ngnix restricts access to only to the last X-Forwarded-For IP address that matches SiteMinder outbound IP address.
6. If allowed, Nginx proxies HTTP request to downstream HTTP server, e.g., WildFly, NodeJS, etc.

In a nutshell, since the inbound client IP address to OpenShift Router is always appended to the X-Forwarded-For AND Nginx trusts OpenShift Router, the IP address may be used for access control decisions. 

### Future and Alternate Considerations

This integration with SiteMinder Web SSO only describes only integration path with IDIM's identity and authentication services.  There also exists SAML 2.0 web flow and hopefully in the future a OAuth2 or OpenIDConnect protocol support.  

Alternative options considered were: a private OpenShift Router, a OpenShift router with the ability firewall. 
  
## NPM Build Process
This S2I will automatically call the following command during the `assemble` phase:

```
npm_build() {
  # run npm install to get dependancies
  npm install /tmp/src/
  # run the build process
  npm run build /tmp/src/
  # copy built software to static folder
  cp -Rf /tmp/src/dist/* ./html
}
```

## Advanced Usage

Don't like the static HTML folder?  Want to remove or add Nginx plugins?  A certbot would be handy? Just want to tinker? 

Required tools install:

1. [Docker Toolbox](https://www.docker.com/products/docker-toolbox)
2. [Make](http://gnuwin32.sourceforge.net/packages/make.htm) (Windows users only)

Steps:

1. Fork this repo (FYI we accept pull requests if the change is done in a way that could be reuseable for others)
2. Clone it
3. `make` it


### Build behavior
There are some environment variables you can set to influence **build** behavior.

`NGINX_STATIC_DIR` sets the repo subdir to use for static files, default
`html`.

Either `NGINX_CONF_FILE` sets a config snippet to use or `NGINX_CONF_DIR`
will copy config from this dir (defaults to `conf.d`).

`NGINX_AUX_DIR` sets the aux directory for auxiliary files.

