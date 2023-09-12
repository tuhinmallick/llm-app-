## LLM App

This repo contains the code for running a LLM App in 2 environments:

1. development: runs locally using docker
2. production: runs on AWS ECS

## Setup Workspace (for new users)

1. Clone the git repo

> from the `llm-app` dir:

2. Create + activate a virtual env:

```sh
python3 -m venv llmenv
source llmenv/bin/activate
```

3. Install `phidata`:

```sh
pip install phidata
```

4. Setup workspace:

```sh
phi ws setup
```

5. Copy `workspace/example_secrets` to `workspace/secrets`:

```sh
cp -r workspace/example_secrets workspace/secrets
```

6. Optional: Create `.env` file:

```sh
cp example.env .env
```

## Run LLM App locally using docker

The `workspace/dev_resources.py` file contains the code for the development resources.

1. Install [docker desktop](https://www.docker.com/products/docker-desktop)

2. Set OpenAI Key

Set the `OPENAI_API_KEY` environment variable using

```sh
export OPENAI_API_KEY=sk-***
```

**OR** set in the `.env` file

3. Start the workspace using:

```sh
phi ws up --env dev --infra docker
```

- Open [localhost:9095](http://localhost:9095) to view the Streamlit App.
- Open [localhost:9090/docs](http://localhost:9090/docs) to view the FastApi docs.
- Open [localhost:8888](http://localhost:8888) to view JupyterLab UI.

## Update python libraries

The `pyproject.toml` file is the [standard](https://peps.python.org/pep-0621/) for configuring python projects. This project is already configured to manage libraries using `pyproject.toml`, which is used to automatically generate the `requirements.txt` file using [pip-tools](https://pip-tools.readthedocs.io/en/latest/).

### To add new python libraries

### Step 1: Update `pyproject.toml`

Open the `pyproject.toml` file and add new libraries to the dependencies section.

### Step 2: Update `requirements.txt`

**Option 1:** Update `requirements.txt` file using a helper script:

```sh
./scripts/upgrade.sh
```

**Option 2:** **OR** Generate the `requirements.txt` file using `pip-compile`:

```sh
pip-compile --no-annotate --pip-args "--no-cache-dir" \
-o requirements.txt \
pyproject.toml
```

### Step 3: Rebuild image and recreate containers

**Option 1:** Rebuild dev images and recreate containers using the `phi` cli

Set `build_images=True` in the `workspace/settings.py` file:

```python
    # -*- Image Settings
    # Repository for images
    image_repo="your-image-repo",
    # Build images locally
    build_images=True,
```

Then force create all dev:docker resources using the `-f` flag:

```sh
phi ws up --env dev --infra docker -f
```

**Option 2:** **OR** Use a helper script to build images

> Update the image repo before running the script

```sh
./scripts/build_dev_image.sh
```

Then restart all dev:docker resources using:

```sh
phi ws restart --env dev --infra docker
```

## Shut down dev environment

Delete dev resources using:

```sh
phi ws down --env dev --infra docker
```

## Run LLM App in production on AWS

The `workspace/prd_resources.py` file contains the code for the production resources. To run in production:

1. Build and push production image
2. Create AWS resources

### Step 1: Build production image

**Option 1:** Build and push production image using the `phi` cli

Update the `image_repo`, `build_images`, `push_images` variables in the `workspace/settings.py` file:

```python
    # -*- Image Settings
    # Repository for images
    image_repo="your-image-repo",
    # Build images locally
    build_images=True,
    # Push images after building
    push_images=True,
```

**NOTE:** If you are using ECR, authenticate with ECR before pushing images.

```sh
aws ecr get-login-password --region [region] | docker login --username AWS --password-stdin [account].dkr.ecr.[region].amazonaws.com
```

Create prd:docker resources using:

```sh
phi ws up --env prd --infra docker
```

**Option 2:** **OR** Use a helper script to build images

> Update the image repo before running the script

```sh
./scripts/build_prd_image.sh
```

### Step 2: Create AWS resources

Create production AWS resources using:

```sh
phi ws up --env prd --infra aws
```

## Update production environment

To update your production environment:

1. Build a new production image
2. Update AWS resources

### Step 1: Rebuild production image

**Option 1:** Build production image using the `phi` cli

**NOTE:** If you are using ECR, authenticate with ECR before pushing images.

```sh
aws ecr get-login-password --region [region] | docker login --username AWS --password-stdin [account].dkr.ecr.[region].amazonaws.com
```

Recreate prd:docker resources using:

```sh
phi ws up --env prd --infra docker -f
```

**Option 2:** **OR** Use a helper script to build images

```sh
./scripts/build_prd_image.sh
```

### Step 2: Update AWS resources

1. If you updated the CPU, Memory or Environment, first update the production task definition

```sh
phi ws patch --env prd --infra aws --name td
```

2. Update production ECS Service

```sh
phi ws patch --env prd --infra aws --name service
```

**Note:** If you **ONLY** want to pick up the new image, you do not need to update the task definition and can only update the service using

```sh
phi ws patch --env prd --infra aws --name service
```

### Shut down production environment

Delete production resources using:

```sh
phi ws down --env prd --infra aws
```
