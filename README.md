# carnap-yyc
Configuration and documentation of deployment of Carnap on UCalgary servers

## Local context

We deploy an installation of Carnap on [RHEL 8](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/) servers operated by the
University of Calgary. The servers themselves live behind a firewall
and communicate with it on TCP port 80. The firewall proxies HTTP traffic
to and from [carnap.ucalgary.ca](https://carnap.ucalgary.ca/) on port
443 (HTTPS) to the
internal server at port 80; so we don't have a need for proxying Carnap or
dealing with SSL.

## Building a Docker image

We use the instructions in the [Carnap Github
README](https://github.com/Carnap/Carnap) to build a
[Docker](https://docs.docker.com/) image. This requires
[Nix](https://nixos.org/learn/) and a *lot* of memory. The UCalgary
servers run RHEL with
[SELinux](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/using_selinux/index),
which [prevents Nix](https://github.com/NixOS/nix/issues/2374) from operating properly. The servers also don't
have enough memory to build Carnap. So we build Carnap on other
machines (specifically, Richard's Ubuntu 25.04 laptop).

Basically, we do this:

- Install and configure Nix:

  ```
  $ bash <(curl -L https://nixos.org/nix/install)
  $ nix-env -iA cachix -f https://cachix.org/api/v1/install
  $ cachix use carnap
  ```

- Build the Docker image:

  ```
  $ nix-build release.nix -A docker -o docker-out
  ```

To test if it works on the build machine:
```
sudo docker image load -i docker-out 
sudo docker run -p 3000:3000 carnap:latest 
```
Carnap should appear at [localhost:3000](http://localhost:3000).

We've published the image on Docker Hub as
[rzach/carnap](https://hub.docker.com/repository/docker/rzach/carnap/general).
(Note that the ["official"
image](https://github.com/Carnap/Carnap/pkgs/container/Carnap%2Fcarnap)
referenced in the [Carnap
Manual](https://carnap.github.io/Carnap-Manual/administration.html#server-setup)
is from July 2022 so no longer up to date.)

## Running Carnap

If you're lucky, you should be able to run Carnap without having to
build it.

- First, install `docker` on your machine.
- Follow the instructions in the [Carnap
  Manual](https://carnap.github.io/Carnap-Manual/administration.html#google-authentication)
  for getting a Google OAuth2 Client ID. Make an authorized redirect
  URI point to `http://localhost:3000/auth/page/google/callback`.
- Fire up a text editor and put the following in a file called, say, `carnap.env`:
  ```
  GOOGLEKEY=<your Google Client ID>
  GOOGLESECRET=<your Google client secret>
  ```
- Run Carnap:
  ```
  $ sudo docker run -p 3000:3000 --volume carnap_data:/data --env-file=carnap.env rzach/carnap:latest
  ```
- Point your browser to [localhost:3000](http://localhost:3000).

This should fire up a Docker container running Carnap with SQLite as
the database provider and all the data saved in the container itself
as a persistent volume (meaning it will be there if you stop and then
start the container again). Follow the instructions for [first-time
setup](https://carnap.github.io/Carnap-Manual/administration.html#post-installation-first-time-setup)
to make yourself an administrator.

Note that the Docker build assumes that all Carnap data lives in
`/data`. It will copy files into the relevant directories (e.g.,
`/data/book` for the Carnap book and `/data/static` for the static
files like webpages, JavaScript, and CSS files). User data will be
stored in a database file in `/data/data` and uploaded files in
`/data/data/documents/`.

Since the data lives inside the Docker container, you don't have access to
it from outside. You can also ask Docker to use an existing folder on
your machine to store its data using a bind mount like so (assuming
your have created the directory `/home/user/carnap-data`):
```
sudo docker run -p 3000:3000 --mount type=bind,src=/home/user/carnap-data/,dst=/data --env-file=carnap.env rzach/carnap:latest
```
## Carnap on carnap.ucalgary.ca

To run Carnap, we use `docker compose` as described in the
[documentation](https://github.com/Carnap/Carnap-Documentation/tree/master/docker-compose-sample).
We use our own `compose.yml` file. It doesn't use Caddy since we
don't need to proxy ports; the firewall does it for us. You need the
files `.env` and `.env-postgres` with actual values (use the example
files from [the documentation Github
repo](https://github.com/Carnap/Carnap-Documentation/tree/master/docker-compose-sample)
as starting points). Once those are set up you can say
```
$ sudo docker compose up -d
```
The `-d` flag "detaches" the process, i.e., it runs in the background
and stays running even if you log out.

