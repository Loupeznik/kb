# Google Container Registry (Artifact Registry)

## Basics

Creating a repository, pushing an image, pulling an image, connecting repository to a Kubernetes cluster outside GCP

### Create a repository

1. Go to `https://console.cloud.google.com/artifacts/browse/<PROJECT_ID>`
2. Create a repository through the wizard
3. Copy the region and the repository name from the created resource

### Connect to the repository and push an image

1. Install the [Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
2. Authenticate with `gcloud auth login`
3. Configure Docker to use `gcloud` as a credential helper: `gcloud auth configure-docker REGION`. For example `gcloud auth configure-docker europe-west3-docker.pkg.dev`
4. Build a Docker image and tag it with the repository name: `docker build -t REGION/PROJECT_ID/REPOSITORY_NAME/IMAGE_NAME:TAG .`. For example `docker build -t europe-west3-docker.pkg.dev/my-project/my-repository/my-image:latest .`
5. Push the image to the repository: `docker push REGION/PROJECT_ID/REPOSITORY_NAME/IMAGE_NAME:TAG`. For example `docker push europe-west3-docker.pkg.dev/my-project/my-repository/my-image:latest`

### Create a service account and connect to Kubernetes (using Helm)

1. Create a service account with the `Artifact Registry Reader` role
2. Create a key for the service account and download it as a JSON file
3. Create a Kubernetes secret in a given namespace

   ```shell
   kubectl create secret -n myapp docker-registry gcr-credentials \
       --docker-server europe-west3-docker.pkg.dev \
       --docker-username _json_key \
       --docker-password DOWNLOADED_JSON_KEY_AS_SINGLE_LINE \
       --docker-email dzarsky@dzarsky.eu
   ```

4. Add the secret reference to the Helm _values.yaml_ file
   ```yaml
   # ...
   imagePullSecrets:
     - name: gcr-credentials
   # ...
   ```
