# Lab 2 - Deploying the api on Google Cloud Run

#### Setting up the GCP project
To execute this step, you should have a google cloud platform account, and a working gcloud command.

First, create a new project on the Google Cloud platform console by logging into the console. Call this project *_cnf-api-production_*.

After your project has been created, you should have something like this: 

<p align="center">
  <img src="../img/gcp_project.png" alt="GCP Console" />
</p>

#### Installing the gcloud command line tool
To interact with Google Cloud Platform resources, you can use the Google Cloud Sdk. It includes 
a command line tool named gcloud.
I suggest you first follow the steps in the [quickstart](https://cloud.google.com/sdk/docs/quickstarts)*[]: 


#### Setting up the gcloud command to interact with our project
Let's run the below commands (1 by 1...) to set up the gcloud command line to be able to take actions on our project:

```bash
gcloud auth login       # Will open a web browser to enable google authentication
gcloud projects list    # Will list the projects associated to your account. Please write down the id (PROJECT_ID) of your project cnf-api-production.


# Setting up gcloud to interact with the cnf-api-production project
export PROJECT_ID=xxx   # The id of the project you just got from the previous step
export REGION=europe-west2-b

gcloud config list      # To see the current configuration of your gcloud command line
gcloud config set project ${PROJECT_ID} # To set the project to interact with
gcloud config set run/region ${REGION} # To set the default region to use for Cloud Run
```

#### Enabling necessary API's
For this project to work properly, we need to activate several api's on Google Cloud Platform:

- [Identity Access Management](https://cloud.google.com/iam/docs/reference/rest) (iam.googleapis.com)
- [Cloud resource manager](https://cloud.google.com/resource-manager/docs/apis) (cloudresourcemanager.googleapis.com): 
- [Google Container Registry](https://cloud.google.com/container-registry/) (containerregistry.googleapis.com)
- [Google Cloud Run](https://cloud.google.com/run/): (run.googleapis.com)

We can do this through the gcp console, or directly with the gcloud command line tool. Let's do it with gcloud (doc [here](https://cloud.google.com/sdk/gcloud/reference/services/enable)):
```bash
# Check the list of available services for our project:
gcloud services list --available

# Activate our 4 necessary apis:
gcloud services enable iam.googleapis.com cloudresourcemanager.googleapis.com containerregistry.googleapis.com run.googleapis.com --async

```

#### Deploying our API
Now that we've enabled the necessary apis, we can push our Docker container to Google Container Registry. It is a private Docker container registry,
working like a private DockerHub registry.


##### Pushing our docker container to Google Container Registry
First, we need to authenticate Docker before pushing the image to our private registry. We can use gcloud for that: 
```bash
gcloud auth configure-docker
```

Then, we need to give a special tag to our Docker image, to tell docker to push it to our private registry. Let's call our image cnf-api-production,
because  will deploy the production environment. The :latest means that it's the latest version. We could also put a semantic version here.
```bash
export DOCKER_TAG=gcr.io/${PROJECT_ID}/cnf-api-production:latest

docker build -t ${DOCKER_TAG} --build-arg config_file=production.ini . # NB: this step is not mandatory if we've already built the image with another tag. We could just add a new tag to the image.

```

And let's just push!
```bash
docker push ${DOCKER_TAG}
```

If you go to the gcp console and check out your images in the cloud container registry, you should see something like this:
<p align="center">
  <img src="../img/gcloud_container_image.gif" alt="GCP Cloud Container Registry" />
</p>

##### Setting up a container on Google Cloud Run
We're almost there, we just now need to deploy our image into a working container. This is where we'll use Google Cloud Run: a serverless architecture to run
docker images.

First, make sure that you've created the file env-vars/production.env, with contents ENV=production
The command
```bash
env_vars=$(cat env-vars/production.env  | paste -sd "," -)
```
will read this file, and extract its contents, separated by commas. This is the format gcloud expects when setting environment variables for a cloud run container.


```bash
export SERVICE_NAME=chuck-norris-facts-api
export ENV_VARS=$(cat env-vars/production.env  | paste -sd "," -)

gcloud beta run deploy ${SERVICE_NAME} \
    --image ${DOCKER_TAG} \
    --set-env-vars=${ENV_VARS} \
    --region europe-west1 \
    --platform managed \
    --allow-unauthenticated \
    --memory 1G
```

And voila! Our API is now available in the cloud, and it's not costing anything when not running. The gcloud command we just ran will
output the url where our api has been deployed. You can test it by executing an http request:
```bash
export API_URL=xxx # the value of the deployed url, returned by the gcloud command.
curl -X GET ${API_URL}/facts/ # will return all facts in our database
```

