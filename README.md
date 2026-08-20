# Jenkins CI/CD — GitHub-triggered Hello World (Poridhi)

A Jenkins setup running in Docker that watches
[`BayajidAlam/jenkins-poc`](https://github.com/BayajidAlam/jenkins-poc)
and runs a multi-stage pipeline on every push:

**Checkout → Build → Test → Package → Deploy**

The Deploy stage serves a static `Hello, World!` page via an `nginx`
container on host port **8088** (8080 is Jenkins, so we use 8088 for
the site). Jenkins and the deployed site are both exposed publicly
via the Poridhi VSCode-proxy URL pattern.

## Files

- `docker-compose.yml` — Jenkins LTS container with the host Docker
  socket mounted (so the Deploy stage can launch nginx).
- `Jenkinsfile` — the pipeline definition. Push this to your repo.
- `index.html` — the static "Hello, World!" page that gets deployed.
  Push this to your repo too.
- `.gitignore` — keeps `.env` (your GitHub PAT) out of the repo.

## 1. Start Jenkins

```bash
cd /home/poridhian/code
docker compose up -d
docker compose logs -f jenkins   # wait for "Jenkins is fully up and running"
```

## 2. Get the initial admin password

```bash
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Paste it into the setup wizard at `http://localhost:8080`, install the
**Suggested Plugins** (includes Git, GitHub, Pipeline), and create
your admin user.

## 3. Expose Jenkins to GitHub via Poridhi

Use the **Poridhi VSCode-proxy URL** (NOT the `lb.poridhi.io` LB UI —
that one serves a static welcome page and doesn't forward paths). The
workspace's proxy URL comes from the `VSCODE_PROXY_URI` env var:

```
https://6932b4db068c684dd55b0c6d_54745ff5.vscode.poridhi.io/proxy/8080/
```

Open it in your browser — you should see the Jenkins login.

## 4. Push the repo

Your repo is currently empty. Push the pipeline + `index.html` so
Jenkins has something to build:

```bash
cd /home/poridhian/code
git init
git add .
git commit -m "Initial Jenkins pipeline + Hello World site"
git branch -M main
git remote add origin https://github.com/BayajidAlam/jenkins-poc.git
# Use your PAT (stored in .env) as the password when prompted,
# or use the token directly in the URL.
git push -u origin main
```

## 5. Create the pipeline job

1. Jenkins → **New Item** → name: `hello-world` → type **Pipeline** → OK.
2. Under **Pipeline**:
   - **Definition**: *Pipeline script from SCM*
   - **SCM**: *Git*
   - **Repository URL**: `https://github.com/BayajidAlam/jenkins-poc.git`
   - **Credentials**: add a GitHub credential (username `BayajidAlam`
     + PAT from `.env` as password), ID `github-pat`.
   - **Branch**: `*/main`
   - **Script Path**: `Jenkinsfile`
3. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**.
4. Save.

## 6. Configure the GitHub webhook

In your GitHub repo → **Settings → Webhooks → Add webhook**:

- **Payload URL**:
  ```
  https://6932b4db068c684dd55b0c6d_54745ff5.vscode.poridhi.io/proxy/8080/github-webhook/
  ```
- **Content type**: `application/json`
- **Which events**: *Just the push event*
- Add webhook.

GitHub will POST a `push` payload to that URL on every push. The Poridhi
VSCode proxy forwards the POST to the Jenkins container's
`/github-webhook/` endpoint, which kicks off a build.

## 7. Test it end-to-end

```bash
cd /home/poridhian/code
echo "Updated at $(date)" >> index.html
git add index.html && git commit -m "Trigger Jenkins" && git push
```

Within seconds you should see a new build in Jenkins with five green
stages (`Checkout`, `Build`, `Test`, `Package`, `Deploy`) and the
`Pipeline completed successfully! 🎉 — Hello World is live.` message.

## 8. View the deployed Hello World

you will see a new build on jenkins:
<img width="1751" height="983" alt="image" src="https://github.com/user-attachments/assets/7c0a39f3-fb1b-4c3b-a119-59d8fe7b0474" />


The Deploy stage launches `nginx` on host port **8088**. View it via
the Poridhi proxy:

```
https://6932b4db068c684dd55b0c6d_54745ff5.vscode.poridhi.io/proxy/8088/
```

You should see the gradient "Hello, World!" page. Each successful push
replaces the running container with the latest `index.html`.

Exponse app though poridhi lb: 

<img width="954" height="506" alt="image" src="https://github.com/user-attachments/assets/d7b06b42-96e0-4702-9016-307dac2b2495" />


<img width="1460" height="919" alt="image" src="https://github.com/user-attachments/assets/27b0f9b7-2a74-4a59-8a67-0906d9f26d4b" />


## Troubleshooting

- **Webhook fails with 404** — you're using the LB URL (`*.lb.poridhi.io`)
  instead of the VSCode-proxy URL (`*.vscode.poridhi.io/proxy/8080/`).
  Re-edit the webhook to use the VSCode proxy URL.
- **Webhook "Invalid HTTP Response"** — usually means Jenkins isn't
  reachable. Confirm `docker compose ps` shows Jenkins `Up` and the
  proxy URL returns 200/403 in your browser.
- **Deploy stage fails (permission denied on docker run)** — make sure
  the Jenkins container runs as root and has `/var/run/docker.sock`
  mounted (already set in `docker-compose.yml`).
- **Port 8088 already in use on host** — change `DEPLOY_PORT` in the
  pipeline's `environment {}` block and the host port mapping in
  `docker-compose.yml` to a free port (e.g. `8089`).

## Tear down

```bash
docker compose down            # stop + remove container, keep volume
docker compose down -v         # also wipe the jenkins_home volume
docker rm -f hello-world-cd    # remove the deployed nginx container
```
