# Kubernetes CI/CD Beginner Lab --- Complete Step-by-Step Record

> **Environment:** Windows, PowerShell in VS Code, Docker Desktop
> Kubernetes, GitHub Actions, Docker Hub\
> **Local project folder:**
> `C:\Users\Mark\Downloads\New folder\student-k8s-demo-1`\
> **Docker Hub image repository:**
> `chaostemptemptemp/student-k8s-demo-1`\
> **Application:** Flask app exposing `/` and `/health` on port `5000`

## Important notes

-   Commands below are intended to be run from the project folder unless
    otherwise stated.
-   Replace placeholders such as `YOUR_GITHUB_USERNAME` with your actual
    values.
-   The successful image tag observed during troubleshooting was
    `bd9a853f016b22f856f76924655a2ae50992e6ee`. **Do not assume this tag
    is still the latest**; use the current commit SHA when updating
    Kubernetes.
-   This record captures the files and troubleshooting steps established
    during the session. Some initial machine setup details (such as
    installing Git, Docker Desktop, VS Code, and creating the remote
    GitHub repository) were not fully recorded in the chat, so those are
    described as prerequisites rather than claimed as completed
    commands.
-   Do not paste Docker Hub access tokens or other secrets into this
    document or commit them to Git.

------------------------------------------------------------------------

## 1. Prerequisites

Install and configure:

1.  Git for Windows
2.  Python 3.12
3.  Docker Desktop, with Kubernetes enabled
4.  Visual Studio Code
5.  `kubectl`
6.  A GitHub repository with a `main` branch
7.  A Docker Hub account and repository named `student-k8s-demo-1`

Check that the tools are available from PowerShell:

``` powershell
git --version
python --version
docker --version
kubectl version --client
```

Check Docker Desktop and Kubernetes:

``` powershell
docker info
kubectl config current-context
kubectl get nodes
```

The Kubernetes context in this lab was Docker Desktop. If Docker Desktop
Kubernetes is not running, start Docker Desktop and enable Kubernetes
before continuing.

## 2. Open the project folder

In VS Code, open:

``` text
C:\Users\Mark\Downloads\New folder\student-k8s-demo-1
```

Or navigate there from PowerShell:

``` powershell
cd "C:\Users\Mark\Downloads\New folder\student-k8s-demo-1"
```

The project should contain the application, tests, Docker files,
workflow, and Kubernetes manifests.

A typical structure is:

``` text
student-k8s-demo-1/
├── app.py
├── test_app.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── .gitignore
├── .github/
│   └── workflows/
│       └── pipeline.yml
└── k8s/
    ├── deployment.yaml
    └── service.yaml
```

## 3. Create the Flask application

Create or update `app.py`:

``` python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello Students! Kubernetes CI/CD is working!"

@app.route("/health")
def health():
    return jsonify(status="healthy"), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### What the application does

-   `GET /` returns a simple message to confirm that the app is
    responding.
-   `GET /health` returns JSON with the health status.
-   `host="0.0.0.0"` allows the application to accept connections from
    outside the container.
-   The app listens on port `5000`.

## 4. Add Python dependencies

Create or update `requirements.txt`:

``` text
Flask>=3.0,<4.0
pytest>=8.0,<9.0
```

Install the dependencies locally:

``` powershell
python -m pip install --upgrade pip
pip install -r requirements.txt
```

## 5. Add automated tests

Create or update `test_app.py`:

``` python
from app import app

def test_home_page():
    client = app.test_client()
    response = client.get("/")
    assert response.status_code == 200
    assert b"Kubernetes CI/CD is working" in response.data

def test_health_endpoint():
    client = app.test_client()
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json["status"] == "healthy"
```

Run the tests:

``` powershell
python -m pytest -v
```

Proceed only when the tests pass.

## 6. Create the Docker image definition

Create or update `Dockerfile` (the filename has no extension):

``` dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

### Add `.dockerignore`

``` text
.venv/
__pycache__/
.pytest_cache/
.git/
```

This prevents unnecessary local files from being sent in the Docker
build context.

### Add `.gitignore`

``` text
.venv/
__pycache__/
.pytest_cache/
.env
```

This helps prevent generated files and local environment files from
being committed.

## 7. Build and test the Docker image locally (recommended check)

Build the image from the project directory:

``` powershell
docker build -t student-k8s-demo-1:local .
```

Run it:

``` powershell
docker run --rm -d --name student-k8s-demo-local -p 5000:5000 student-k8s-demo-1:local
```

Test the application in a browser:

-   `http://localhost:5000/`
-   `http://localhost:5000/health`

Or test from PowerShell:

``` powershell
Invoke-WebRequest http://localhost:5000/
Invoke-RestMethod http://localhost:5000/health
```

Stop the local container when finished:

``` powershell
docker stop student-k8s-demo-local
```

If port `5000` is already in use, stop the process/container using it or
map a different host port, for example `-p 5001:5000`.

## 8. Configure the GitHub Actions workflow

Create or update `.github/workflows/pipeline.yml`:

``` yaml
name: Student Kubernetes CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    name: Checkout and Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests
        run: python -m pytest -v

  docker:
    name: Build and Push Docker Image
    needs: test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/student-k8s-demo-1:${{ github.sha }}
```

### Configure GitHub Actions secrets

In the GitHub repository:

1.  Open **Settings**.
2.  Open **Secrets and variables → Actions**.
3.  Add a repository secret named `DOCKERHUB_USERNAME` with the Docker
    Hub username.
4.  Add a repository secret named `DOCKERHUB_TOKEN` with a Docker Hub
    access token.

For this lab, the Docker Hub username was `chaostemptemptemp`, and the
Docker Hub repository was `student-k8s-demo-1`.

**Repository-name correction made during troubleshooting:** the
workflow's Docker image path was changed from `student-k8s-demo` to
`student-k8s-demo-1`, so the image tag matches the existing Docker Hub
repository.

The workflow builds and pushes an image tagged with the Git commit SHA,
but only on a push to `main`. Pull requests run the test job without
publishing an image.

## 9. Commit and push the application/workflow

Check the repository state:

``` powershell
git status
```

Stage and commit the files:

``` powershell
git add .
git commit -m "Set up Kubernetes CI/CD demo"
```

Push to the `main` branch:

``` powershell
git push origin main
```

If Git says there is nothing to commit, that can be normal if all
changes were already committed.

### Verify the workflow

1.  Open the GitHub repository in a browser.
2.  Open the **Actions** tab.
3.  Open the latest run for **Student Kubernetes CI/CD**.
4.  Confirm that **Checkout and Test** succeeds.
5.  For a push to `main`, confirm that **Build and Push Docker Image**
    succeeds.
6.  Open Docker Hub and verify that the `student-k8s-demo-1` repository
    contains a tag matching the commit SHA.

To get the latest local commit SHA:

``` powershell
git rev-parse HEAD
```

Keep that SHA for the Kubernetes deployment image.

## 10. Create the Kubernetes Deployment

Create the folder if it does not exist:

``` powershell
New-Item -ItemType Directory -Force .\k8s
```

Create or update `k8s/deployment.yaml`. Use the current commit SHA in
place of `REPLACE_WITH_COMMIT_SHA`.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: student-k8s-demo-1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: student-k8s-demo-1
  template:
    metadata:
      labels:
        app: student-k8s-demo-1
    spec:
      containers:
        - name: student-k8s-demo-1
          image: chaostemptemptemp/student-k8s-demo-1:REPLACE_WITH_COMMIT_SHA
          ports:
            - containerPort: 5000
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 5
```

**Important:** The image tag must exist in Docker Hub. The previously
observed successful tag during this session was:

``` text
bd9a853f016b22f856f76924655a2ae50992e6ee
```

That tag was successfully pulled in the cluster at the time. For a later
code change, push the new commit, wait for GitHub Actions to publish it,
and update the manifest to the new SHA.

Apply the Deployment:

``` powershell
kubectl apply -f .\k8s\deployment.yaml
```

Check the Deployment and pods:

``` powershell
kubectl get deployments
kubectl get pods --show-labels
kubectl get pods -l app=student-k8s-demo-1
```

The working Deployment observed during this session was
`student-k8s-demo-1`, with two pods labelled `app=student-k8s-demo-1`,
both in `Running` and `1/1 Ready` state.

## 11. Create the Kubernetes Service

Create or update `k8s/service.yaml`:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: student-k8s-demo
spec:
  type: NodePort
  selector:
    app: student-k8s-demo-1
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 30080
```

### Why the selector matters

-   The Service name is `student-k8s-demo`.
-   The healthy Deployment and its pods are labelled
    `app=student-k8s-demo-1`.
-   A Service's `spec.selector` must match the labels on the pods it
    should route traffic to.
-   The Service accepts traffic on port `80` and forwards it to
    container port `5000`.
-   NodePort `30080` is configured, although the local port-forward
    method below is the recommended verification step in this record.

Apply the Service:

``` powershell
kubectl apply -f .\k8s\service.yaml
```

## 12. Troubleshooting performed: Service had no endpoints

### Symptoms

The Service existed but its endpoints were empty:

``` text
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
student-k8s-demo   NodePort    10.96.113.187   <none>        80:30080/TCP   13d
```

The endpoint check showed no addresses:

``` text
NAME               ENDPOINTS   AGE
student-k8s-demo               13d
```

Inspection showed two Deployments:

``` text
NAME                 READY   UP-TO-DATE   AVAILABLE
student-k8s-demo     0/2     1            0
student-k8s-demo-1   2/2     2            2
```

The healthy pods had this label:

``` text
app=student-k8s-demo-1
```

The Service was still selecting:

``` text
app=student-k8s-demo
```

That mismatch caused the empty endpoints.

### Patch command used successfully

To avoid PowerShell JSON-quoting problems, a YAML patch file was used.

Create `service-patch.yaml` in the project folder with:

``` yaml
spec:
  selector:
    app: student-k8s-demo-1
```

Apply it:

``` powershell
kubectl patch service student-k8s-demo --type=merge --patch-file .\service-patch.yaml
```

This changes the Service selector; it does not delete or restart the
healthy pods.

Verify:

``` powershell
kubectl get endpoints student-k8s-demo
```

The successful result was:

``` text
NAME               ENDPOINTS                           AGE
student-k8s-demo   10.244.0.10:5000,10.244.0.11:5000   13d
```

The two IP addresses and port `5000` confirmed that the Service now
found both healthy pods. Kubernetes may print a deprecation warning for
the legacy Endpoints resource; that warning alone does not indicate
failure.

### Why the earlier patch commands failed

The first JSON patch attempt failed with:

``` text
Error from server (BadRequest): invalid character 's' looking for beginning of object key string
```

A second attempt also failed to decode the patch. The YAML patch file
method above worked and is the known-good method from this session.

## 13. Verify the Service and pods

Run:

``` powershell
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get endpoints student-k8s-demo
kubectl describe service student-k8s-demo
```

Confirm:

-   Deployment `student-k8s-demo-1` has `2/2` available.
-   Both `student-k8s-demo-1` pods are `Running` and `1/1` ready.
-   Service `student-k8s-demo` has NodePort `30080`.
-   Service endpoints list two pod IP addresses on port `5000`.
-   The Service selector is `app=student-k8s-demo-1`.

The old `student-k8s-demo` Deployment and its image-pull-failing pods
were still present at the last recorded check. **Do not delete them
until you have verified the working application and confirmed that no
other lab requirement depends on them.**

## 14. Test the application using port-forward

Run:

``` powershell
kubectl port-forward service/student-k8s-demo 8080:80
```

Keep this terminal open. Expected output is similar to:

``` text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Open these URLs in your browser:

-   Main page: `http://localhost:8080/`
-   Health check: `http://localhost:8080/health`

Expected main-page text:

``` text
Hello Students! Kubernetes CI/CD is working!
```

Expected health response:

``` json
{"status":"healthy"}
```

You can also test from a **second PowerShell terminal** while the
port-forward process is running:

``` powershell
Invoke-WebRequest http://localhost:8080/
Invoke-RestMethod http://localhost:8080/health
```

Stop port-forwarding when finished by returning to its terminal and
pressing:

``` text
Ctrl + C
```

## 15. Updating the application after a code change

For each future code change:

1.  Edit the application or tests.

2.  Run the tests locally:

    ``` powershell
    python -m pytest -v
    ```

3.  Commit and push:

    ``` powershell
    git add .
    git commit -m "Describe the change"
    git push origin main
    ```

4.  Wait for GitHub Actions to pass and publish the Docker image.

5.  Get the current commit SHA:

    ``` powershell
    git rev-parse HEAD
    ```

6.  Update the image tag in `k8s/deployment.yaml` to:

    ``` text
    chaostemptemptemp/student-k8s-demo-1:CURRENT_COMMIT_SHA
    ```

    Replace `CURRENT_COMMIT_SHA` with the actual SHA. Do not type the
    literal placeholder.

7.  Apply the Deployment update:

    ``` powershell
    kubectl apply -f .\k8s\deployment.yaml
    ```

8.  Watch the rollout:

    ``` powershell
    kubectl rollout status deployment/student-k8s-demo-1
    kubectl get pods -l app=student-k8s-demo-1
    ```

9.  Verify Service endpoints and test the app again:

    ``` powershell
    kubectl get endpoints student-k8s-demo
    kubectl port-forward service/student-k8s-demo 8080:80
    ```

## 16. Common problems and checks

### `ImagePullBackOff` or `ErrImagePull`

The cluster cannot pull the image. Check the Deployment's image name and
tag:

``` powershell
kubectl describe pod POD_NAME
kubectl describe deployment student-k8s-demo-1
```

Replace `POD_NAME` with the actual pod name. Verify that the exact
image/tag exists in Docker Hub and that the repository name is correct.

### Service has no endpoints

Check labels and selector:

``` powershell
kubectl get pods --show-labels
kubectl describe service student-k8s-demo
kubectl get endpoints student-k8s-demo
```

The Service selector must match the labels on ready pods.

### Application is not ready

Check the rollout and pod logs:

``` powershell
kubectl rollout status deployment/student-k8s-demo-1
kubectl logs deployment/student-k8s-demo-1
kubectl describe pods -l app=student-k8s-demo-1
```

The readiness probe expects `/health` to return HTTP `200` on port
`5000`.

### Port-forward cannot start

Check whether the port-forward command is still running in another
terminal, and whether port `8080` is already occupied. If necessary, use
another local port:

``` powershell
kubectl port-forward service/student-k8s-demo 8081:80
```

Then visit `http://localhost:8081/`.

### GitHub Actions succeeds but Kubernetes uses an old image

The workflow tags images with the commit SHA. Ensure the manifest uses
the SHA that was just pushed, then reapply it:

``` powershell
kubectl apply -f .\k8s\deployment.yaml
kubectl rollout status deployment/student-k8s-demo-1
```

------------------------------------------------------------------------

## Final checklist

-   [ ] Flask app responds at `/`.
-   [ ] `/health` returns `{"status":"healthy"}`.
-   [ ] `python -m pytest -v` passes.
-   [ ] Docker image builds successfully.
-   [ ] GitHub Actions test job passes.
-   [ ] Docker build-and-push job passes on a push to `main`.
-   [ ] Correct image tag exists in Docker Hub.
-   [ ] Deployment `student-k8s-demo-1` has two ready pods.
-   [ ] Service selector is `app=student-k8s-demo-1`.
-   [ ] Service endpoints show two pod IPs on port `5000`.
-   [ ] Browser test at `http://localhost:8080/` displays the expected
    message.
-   [ ] Browser test at `http://localhost:8080/health` returns healthy
    JSON.

**End state recorded in this session:** the two pods for
`student-k8s-demo-1` were running, and the Service endpoint check showed
`10.244.0.10:5000` and `10.244.0.11:5000`. The port-forward/browser test
was the next step; its result had not yet been reported.
