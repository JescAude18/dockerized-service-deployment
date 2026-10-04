<a href="https://gitmoji.dev">
  <img
    src="https://img.shields.io/badge/gitmoji-%20😜%20😍-FFDD67.svg?style=flat-square"
    alt="Gitmoji"
  />
</a>

# Dockerized Service Deployment

Use GitHub Actions to Deploy a Dockerized Node.js Service

**Project Reference:** [roadmap.sh/projects/dockerized-service-deployment](https://roadmap.sh/projects/dockerized-service-deployment)

## Table of Contents

- [About](#about)
- [Features](#features)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation & Usage](#installation--usage)
- [Example Output](#example-output)
- [How It Works](#how-it-works)
- [Error Handling](#error-handling)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Author](#author)
- [License](#license)

## About

This project is a small Node.js HTTP service packaged as a Docker image and
deployed to a remote Linux virtual machine.

The service exposes two public behaviors:

- `GET /` returns a plain-text greeting.
- `GET /secret` returns a secret message only when valid HTTP Basic
  Authentication credentials are provided.

The repository also contains the infrastructure and automation used to prepare
and deploy the service:

- Terraform provisions an Oracle Cloud Infrastructure (OCI) compute instance.
- Ansible updates the server and installs Docker and supporting utilities.
- GitHub Actions builds and publishes the image, then connects to the VM over
  SSH and runs the container.

Two deployment workflows are available. They perform the same deployment flow
but publish the image to different registries:

- Docker Hub: `.github/workflows/deploy_with_dockerhub.yaml`
- GitHub Container Registry: `.github/workflows/deploy_with_ghcr.yaml`

## Features

- Minimal Node.js HTTP server using the built-in `node:http` module.
- Docker image based on `node:22-alpine`.
- Runtime configuration through `APP_USERNAME`, `APP_PASSWORD`, and
  `SECRET_MESSAGE`.
- Basic Authentication for the `/secret` endpoint.
- Immutable image tags based on the Git commit SHA.
- Docker Buildx-based image build and push.
- Deployment over SSH with `appleboy/ssh-action`.
- Docker Hub and GHCR deployment alternatives.
- OCI VM provisioning with Terraform.
- Server bootstrapping with Ansible roles.
- Existing application containers are replaced during deployment.

## Project Structure

```text
.
├── server.js                         # Node.js HTTP service
├── package.json                      # npm metadata and start script
├── Dockerfile                        # Container image definition
├── .dockerignore                     # Files excluded from the Docker build context
├── ansible/
│   ├── inventory.ini                 # Target web server connection data
│   ├── setup.yaml                    # Entry point for server configuration
│   └── roles/
│       ├── base/tasks/main.yaml      # OS updates and base utilities
│       └── docker/tasks/main.yaml    # Docker repository, packages, and service
├── terraform/
│   ├── provider.tf                   # OCI provider configuration
│   ├── variables.tf                  # Required infrastructure inputs
│   ├── data.tf                       # OCI availability-domain lookup
│   └── main.tf                       # OCI compute instance and public VNIC
└── .github/workflows/
    ├── deploy_with_dockerhub.yaml    # Build, push, pull, and deploy via Docker Hub
    └── deploy_with_ghcr.yaml         # Build, push, pull, and deploy via GHCR
```

Terraform variable values are supplied through Terraform configuration files
that are intentionally ignored by Git. The repository also ignores `.env`,
Terraform state files, and `node_modules`.

## Requirements

### Local application development

- Node.js 22 or a compatible newer Node.js runtime.
- npm.
- Docker.

### Infrastructure and deployment

- An OCI account and an OCI API key for Terraform.
- Terraform with the OCI provider.
- An existing OCI subnet and image OCID.
- An SSH key pair for the VM.
- An Ansible installation.
- A remote Ubuntu-like Linux VM accessible over SSH.
- A Docker Hub repository, or a GHCR package associated with the repository.
- GitHub repository variables and secrets required by the selected workflow.

The workflows expect these GitHub values:

| Name | Type | Used for |
| --- | --- | --- |
| `DOCKERHUB_USERNAME` | Repository variable | Docker Hub login |
| `DOCKERHUB_TOKEN` | Repository secret | Docker Hub authentication |
| `SERVER_IP` | Repository variable | Remote VM address |
| `SERVER_USER` | Repository variable | SSH user on the VM |
| `SSH_PRIVATE_KEY` | Repository secret | SSH authentication |
| `APP_USERNAME` | Repository secret | Valid `/secret` username |
| `APP_PASSWORD` | Repository secret | Valid `/secret` password |
| `SECRET_MESSAGE` | Repository secret | Message returned by `/secret` |

The Docker Hub workflow uses the first two Docker Hub values. The GHCR
workflow instead uses the workflow-provided `GITHUB_TOKEN` and repository
identity for registry authentication.

## Installation & Usage

### Clone or download the repository

Clone the repository with Git and move into the project directory:

```bash
git clone https://github.com/JescAude18/dockerized-service-deployment.git
cd dockerized-service-deployment
```

If Git is not available, download the repository as a ZIP archive from
[GitHub](https://github.com/JescAude18/dockerized-service-deployment), extract
it, and open a terminal in the extracted `dockerized-service-deployment`
directory.

### Run the service locally with Node.js

Install dependencies and start the server:

```bash
npm install
npm start
```

The server listens on port `3000`:

```bash
curl http://localhost:3000/
curl -u "$APP_USERNAME:$APP_PASSWORD" http://localhost:3000/secret
```

For `/secret`, define `APP_USERNAME`, `APP_PASSWORD`, and `SECRET_MESSAGE` in
the process environment before starting the application. The application reads
these values at request time through `process.env`.

### Build and run the Docker image locally

Build the image and start a container:

```bash
docker build -t dockerized-nodejs-service .

docker run -d \
  --name dockerized-nodejs-service \
  -e APP_USERNAME="$APP_USERNAME" \
  -e APP_PASSWORD="$APP_PASSWORD" \
  -e SECRET_MESSAGE="$SECRET_MESSAGE" \
  -p 3000:3000 \
  dockerized-nodejs-service
```

The Dockerfile installs the dependencies, copies the application, exposes
port `3000`, and starts the service with `npm start`.

### Provision the OCI VM with Terraform

Create a local Terraform variable file containing the values declared in
`terraform/variables.tf`, then initialize and apply Terraform:

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

The Terraform configuration creates an OCI compute instance with a public IP,
using an existing subnet and image. It also adds the configured public SSH key
to the instance.

Do not commit OCI credentials, private key paths, Terraform variable files, or
Terraform state. These files are ignored by the repository.

### Configure the VM with Ansible

Update `ansible/inventory.ini` with the VM address, SSH user, and private key
path, then run:

```bash
ansible-playbook -i ansible/inventory.ini ansible/setup.yaml
```

The playbook targets the `webservers` group. It applies the `base` role first,
which updates the system and installs utilities, then applies the `docker`
role, which adds Docker's Ubuntu repository, installs Docker Engine packages,
and starts and enables the Docker service.

### Deploy with GitHub Actions

Both workflows run when code is pushed to `main` and can also be started
manually with `workflow_dispatch`.

Select one workflow in the GitHub Actions interface:

1. The `prepare_image` job logs in to the selected registry.
2. Docker Buildx builds the image and pushes it with the current
   `github.sha` tag.
3. The `deploy` job waits for `prepare_image` to succeed.
4. An SSH session logs in to the registry on the VM and pulls that exact tag.
5. A second SSH session removes the previous container and starts the new one.

The container is named `dockerized-nodejs-service` and maps VM port `3000` to
container port `3000`.

## Example Output

The root endpoint returns:

```text
$ curl http://localhost:3000/
Hello, world!
```

With valid Basic Authentication, the secret endpoint returns the configured
runtime message:

```text
$ curl -u app-user:app-password http://localhost:3000/secret
This message comes from the application configuration.
```

The exact secret response depends on the value of `SECRET_MESSAGE`.

Typical HTTP responses are:

| Request | Status | Response |
| --- | ---: | --- |
| `GET /` | `200` | `Hello, world!` |
| `GET /secret` with valid Basic Auth | `200` | `SECRET_MESSAGE` |
| `GET /secret` without credentials | `401` | `Authentication not supported!` |
| `GET /secret` with unsupported auth type | `401` | `Authentication not supported!` |
| `GET /secret` with invalid credentials | `401` | `Unauthorized!` |
| Any other path | `404` | `Not found!` |

## How It Works

### Application

`server.js` creates an HTTP server that listens on port `3000`. It checks the
request URL and sends a plain-text response:

- `/` is public.
- `/secret` expects an `Authorization` header using the `Basic` scheme.
- Any other route returns `404`.

For Basic Authentication, the server decodes the Base64 credentials, separates
the username and password at `:`, and compares them with `APP_USERNAME` and
`APP_PASSWORD`. A successful comparison returns `SECRET_MESSAGE`.

### Container image

The Dockerfile uses `node:22-alpine`, sets `/app` as the working directory,
copies the npm manifest files, runs `npm install`, copies the source code, and
starts the application with `npm start`.

`.dockerignore` excludes `.env`, Git metadata, GitHub workflows, infrastructure
directories, and documentation from the build context. Application secrets are
therefore supplied at runtime rather than baked into the image.

### CI/CD image flow

Each workflow uses the commit SHA as the image tag:

```text
GitHub Actions runner
  -> build image
  -> push registry/image:<github.sha>
  -> SSH to VM
  -> docker pull registry/image:<github.sha>
  -> replace running container
```

The Docker Hub workflow authenticates with `DOCKERHUB_USERNAME` and
`DOCKERHUB_TOKEN`. The GHCR workflow authenticates with `github.actor` and
`GITHUB_TOKEN` and declares `contents: read` and `packages: write`
permissions.

The workflow-level `IMAGE_NAME` exists in the GitHub Actions workflow context.
The SSH actions explicitly list variables in `envs` so that they become
available in the remote shell. The `docker run -e` options then pass the
application configuration from that remote shell into the container.

### Infrastructure flow

Terraform creates the OCI compute instance and assigns a public IP through a
VNIC. Ansible prepares the operating system and installs Docker. GitHub
Actions subsequently treats that host as the deployment target.

## Error Handling

The Node.js service reports errors through HTTP status codes rather than a
framework-level error handler:

- Missing or unsupported authentication on `/secret` returns `401`.
- Incorrect credentials return `401`.
- Unknown routes return `404`.

The deployment script removes the previous container with:

```bash
sudo docker rm -f dockerized-nodejs-service || true
```

The `|| true` makes the removal step succeed when no previous container exists,
which supports a first deployment. It does not replace error handling for the
image pull, container start, SSH connection, or registry login: those commands
must still succeed for the workflow step to complete successfully.

## Roadmap

The following items are not currently implemented, but are natural next steps
for this project:

- Add automated unit and HTTP integration tests.
- Add a health check and deployment verification after starting the container.
- Add a reverse proxy and HTTPS termination for the public service.
- Improve authentication handling and validate malformed Basic Auth headers.
- Use a production-oriented dependency installation strategy such as
  `npm ci` with a committed lockfile.
- Add image vulnerability scanning and workflow status checks.
- Add rollback support to a previously deployed image tag.
- Move Terraform and Ansible inputs to a documented, secret-safe example
  configuration.

## Contributing

1. Create a feature branch.
2. Keep application, infrastructure, and workflow changes focused.
3. Do not commit `.env` files, credentials, private keys, Terraform state, or
   other secrets.
4. Run the relevant local commands before opening a pull request.
5. Document changes that affect routes, environment variables, infrastructure,
   or deployment configuration.

Pull requests should explain how the change was tested and whether it affects
the Docker image or either deployment workflow.

## Author

**Created by**: Jessica MOUSSOUGAN

**Email**: [jessicamoussougan@gmail.com](mailto:jessicamoussougan@gmail.com)

**GitHub**: [@JescAude18](https://github.com/JescAude18)

## License

No license yet.

This project is currently for personal training and learning.
