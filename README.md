# Jenkins CI/CD — GitHub-triggered Hello World

A Jenkins setup running in Docker that watches
[`BayajidAlam/jenkins-poc`](https://github.com/BayajidAlam/jenkins-poc)
and runs a multi-stage pipeline on every push:

**Build → Test → Package → Deploy**

The deploy stage serves a static `Hello, World!` page via an `nginx`
container on port 80 of the Jenkins host (exposed via the Poridhi proxy).

## Files

- `docker-compose.yml` — Jenkins LTS container with a persistent volume
  and the host Docker socket mounted (so the Deploy stage can launch
  nginx).
- `Jenkinsfile` — the pipeline definition. Push this to your repo.
- `index.html` — the static "Hello, World!" page that gets deployed.
  Push this to your repo too.

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
**Suggested Plugins** (this includes the Git, GitHub, and Pipeline
plugins), and create your admin user.

## 3. Expose Jenkins to GitHub via Poridhi

GitHub needs a public URL to send the webhook to. This workspace's
Poridhi proxy URL is built from the `VSCODE_PROXY_URI` env var.
Replace `{{port}}` with `8080` (Jenkins web UI port):

```
https://6932b4db068c684dd55b0c6d_54745ff5.vscode.poridhi.io/proxy/8080/
```

Test it in your browser — you should see the Jenkins login page.

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
git push -u origin main
```

## 5. Create the pipeline job

1. Jenkins → **New Item** → name it `hello-world` → type **Pipeline** → OK.
2. Under **Pipeline**:
   - **Definition**: *Pipeline script from SCM*
   - **SCM**: *Git*
   - **Repository URL**: `https://github.com/BayajidAlam/jenkins-poc.git`
   - **Credentials**: add a GitHub credential (username + PAT) if the
     repo is private. Leave blank if it's public.
   - **Branch**: `*/main`
   - **Script Path**: `Jenkinsfile`
3. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**.
4. Save.

## 6. Configure the GitHub webhook

In your GitHub repo:

1. **Settings** → **Webhooks** → **Add webhook**.
2. **Payload URL**:
   ```
   https://6932b4db068c684dd55b0c6d_54745ff5.vscode.poridhi.io/proxy/8080/github-webhook/
   ```
   (Note the trailing `/github-webhook/` — that's the endpoint the
   Jenkins GitHub plugin listens on.)
3. **Content type**: `application/json`.
4. **Which events**: *Just the push event.*
5. Add webhook.

## 7. Test it end-to-end

Make a change and push:

```bash
cd /home/poridhian/code
echo "Updated at $(date)" >> index.html
git add index.html
git commit -m "Trigger Jenkins"
git push origin main
```

Within seconds you should see a new build in Jenkins with four green
stages (`Build`, `Test`, `Package`, `Deploy`) and the
`Pipeline completed successfully! 🎉 — Hello World is live.` message.

## 8. View the deployed Hello World

The Deploy stage launches `nginx` on port 80 of the Jenkins host.
Open it via the Poridhi proxy:

```
https://6932b4db068c684dd55b0c6d_54745ff5.vscode.poridhi.io/proxy/80/
```

You should see the gradient "Hello, World!" page. Each successful push
replaces the running container with the latest `index.html`.

## Troubleshooting

- **Webhook not firing?** Check
  **GitHub → Settings → Webhooks → Recent Deliveries** — red icons
  show the network call failed. Make sure the URL ends in
  `/github-webhook/`.
- **Deploy stage fails?** The Jenkins container needs `/var/run/docker.sock`
  mounted (already done in `docker-compose.yml`) and should run as
  `root` (already set). Check `docker compose logs jenkins` for the
  actual error.
- **Port 80 already in use?** Stop the conflicting container, or
  change `- "80:80"` in `docker-compose.yml` to a different host port
  (e.g. `- "8888:80"`) and update the deploy step's `-p 80:80` to
  match.

## Tear down

```bash
docker compose down            # stop + remove container, keep volume
docker compose down -v         # also wipe the jenkins_home volume
docker rm -f hello-world-cd    # remove the deployed nginx container
```
