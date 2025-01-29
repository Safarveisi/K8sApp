# Kubernetes microservices

Here's an example of deploying various services on a Kubernetes cluster, such as a FastAPI web service and a Streamlit dashboard. This setup utilizes an Nginx controller as an ingress resource to route traffic to the appropriate service based on the originating endpoint. You can refer to `/deploy/k8s/common/ingress.yml` for more details. 

## Prerequisites :hammer_and_wrench:

To get started, you'll need access to a Kubernetes cluster. For this project, I used a managed Kubernetes cluster on IONOS Cloud, provisioned using Terraform (refer to `/terraform` for more information). Additionally, you should install `devbox` and activate its shell to access the necessary software packages for this project (see `devbox.json`). Lastly, ensure that you have both the `Docker Engine` and `pyenv` installed. Using `pyenv`, install a Python version that is compatible with `pyproject.toml` file, such as Python 3.9.1, used in this project.

### Provision a Kubernetes cluster
Activate the `devbox` shell (run `devbox shell`), navigate to `/terraform` directory and run

```bash
terraform plan -out="tfplan"
terraform apply "tfplan"
```

It will take some time to create the resources. 

> [!Note]
> To create a Kubernetes cluster, you can also use other Terraform providers. If you decide to do so,
> you'll need to update `main.tf` or any other files with the `.tf` extension in `/terraform`. You can visit
> [ionos-cloud](https://registry.terraform.io/providers/ionos-cloud/ionoscloud/latest/docs) for more information 
> about the current provider. Additionally, if you opt for a different provider, the `helper/get_k8s_config.py`
> script might no longer be necessary.

### Setup the python virtual env
In the project root directory, run

```
./setup-env
```

You can now acquire the provisioned Kubernetes cluster's config file by running

```bash
poetry run python helper/get_k8s_config.py 
```

This will save `config` file in `~/.kube/` (note that existing `config` file will be replaced). You can modify the python module to suit your needs (e.g., put the `config` file in an arbitrary location). Both `kubectl` and `k9s` will search for the config file in this location to authenticate themselves with our Kubernetes cluster. 

> [!Caution]
> Make sure the `config` file is not tracked by git. 

## Usage :unlock:

:one: Navigate to the following directories one by one and use `task build-container-image-multi-arch` to build a multi arch image and push it to `docker.io` registery.

* `deploy/k8s/postgres`
* `deploy/k8s/apps/python-api`
* `deploy/k8s/apps/streamlit`

We aimed to follow best practices when writing the Dockerfiles.

> [!Note]
> Make sure you have a builder instance with driver of type docker-container. Otherwise, `docker buildx build` fails. 

:two: Navigate to `deploy/k8s` directory and create the Kubernetes resources by running

```bash
task apply-all
``` 

Again, we adhered to best practices when creating the manifest files for deploying the apps (python-api and Streamlit).

> [!Note]
> You can also navigate to the directories listed above and execute tasks manually. To get the list of tasks, run `tl` (an alias for `task --list`) . You can then execute a task by `task <task-name>`

You can use `k9s` to see the status of deployments and the pods' log. 

:three: Add the external ip of the load balancer to `/etc/hosts` of your machine and use `ionos.ingress-nginx.com` as for the the hostname. You can use a different hostname, but you need to make sure to modify the manifest files where needed. 

To get the external ip address, run

```bash
kubectl get svc -n ingress-nginx
```

:sunglasses: You can now make an API request to http://ionos.ingress-nginx.com/fast/python-api/ with the parameter `select-api?api_name=node`.
FastAPI will handle the request and store the request parameter, `api_name=node`, along with its timestamp, in a PostgreSQL table (public.request). Additionally, you can access the `Streamlit` dashboard by navigating to http://ionos.ingress-nginx.com/streamlit/.

## Continuous integration (CI)

We utilize GitHub Actions for continuous integration, as specified in the `.github/workflows` directory. 
Essentially, the workflow is triggered whenever a new Git tag is created. This Git tag is applied to all Docker images, so if the new tag is 1.0.0, all Docker images will carry the same tag. Additionally, a pull request is initiated to merge these changes into the master branch.

> [!Note]
> You need to replace the secrets used in the workflows with yours (e.g., `DOCKERHUB_TOKEN`). 

## System :globe_with_meridians:

![Diagram of components](./pictures/diagram.png)
