# Running FaaSr on the National Research Platform (NRP)

This is a step-by-step walkthrough for running FaaSr workflows on the [National Research Platform (NRP)](https://nationalresearchplatform.org/) — the **Nautilus** Kubernetes cluster — from an empty machine to a workflow whose actions run as Kubernetes Jobs on NRP.

NRP/Nautilus is a **single shared, multi-institution Kubernetes cluster**. You do **not** create your own cluster — you are granted a **namespace** inside Nautilus and run your Jobs there. For the Kubernetes compute-server field reference, see [Running on Kubernetes]. This page focuses on the NRP-specific setup.

## 1. Get NRP access

1. Go to the [NRP portal](https://nrp.ai) and **Log In** — authentication is via CILogon; pick your institution.
2. Get access to a **namespace**:
    - **Students:** ask your advisor to add you to their namespace.
    - **Faculty/staff/postdocs:** request a namespace via Nautilus support (the [Matrix/Slack support channels](https://nationalresearchplatform.org/)).
3. Read and follow the [NRP Acceptable Use Policy and cluster policies](https://docs.nationalresearchplatform.org/). One rule matters for FaaSr:

    !!! warning "No never-ending Jobs"
        NRP bans workloads that run forever (a `sleep`, an idle server, etc.). FaaSr actions run a function and exit on their own, so normal workflows are fine — just don't set an action to block indefinitely.

## 2. Install the CLI tools

You need **`kubectl`** and, for NRP specifically, the **`kubelogin`** OIDC plugin (without it the NRP kubeconfig will not authenticate).

```bash
# kubectl (Linux amd64; see kubernetes.io for other platforms)
curl -fsSL -o ~/.local/bin/kubectl \
  "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ~/.local/bin/kubectl

# kubelogin, installed under the name kubectl expects (kubectl-oidc_login)
KVER=$(curl -fsSL https://api.github.com/repos/int128/kubelogin/releases/latest | jq -r .tag_name)
curl -fsSL -o /tmp/kubelogin.zip \
  "https://github.com/int128/kubelogin/releases/download/${KVER}/kubelogin_linux_amd64.zip"
unzip -o /tmp/kubelogin.zip kubelogin -d /tmp/
mv /tmp/kubelogin ~/.local/bin/kubectl-oidc_login
chmod +x ~/.local/bin/kubectl-oidc_login
```

Make sure `~/.local/bin` is on your `PATH` (or install to `/usr/local/bin` with `sudo`).

## 3. Download your kubeconfig

NRP serves a ready-made kubeconfig that uses OIDC:

```bash
mkdir -p ~/.kube
curl -o ~/.kube/config -fSL "https://nrp.ai/config"
```

This sets the context `nautilus`, embeds the cluster's API endpoint and CA certificate, and wires up the `oidc-login` exec plugin.

## 4. Authenticate

The first `kubectl` command triggers a browser login through CILogon:

```bash
kubectl get pods -n <your-namespace>
```

- `No resources found in <your-namespace> namespace` means you're authenticated and **have access** to the namespace (there are just no pods yet).
- Your OIDC token lasts ~30 minutes and auto-refreshes; `kubectl oidc-login clean` forces a fresh login (useful right after you're added to a new namespace).

!!! tip "Multi-account browser gotcha"
    If you're signed into several Google/GitHub accounts in your default browser, CILogon may auto-select the wrong identity and you'll see **"Permission denied — authenticate with the source you've previously signed up with."** The fix is to make the login open in a **fresh incognito window**. Either open the printed device-code URL in incognito, or point kubelogin at an incognito browser by adding these args to the `oidc-login` block in `~/.kube/config`:

    ```yaml
    - --browser-command=/path/to/incognito.sh   # a script that runs: google-chrome --incognito "$@"
    ```

    (or add `--grant-type=device-code` and `--skip-open-browser`, then open the URL yourself in incognito).

## 5. Create a service-account token (the JWT FaaSr uses)

Your OIDC login authenticates **you** interactively and expires quickly — that can't be used by a GitHub Action. FaaSr instead needs a long-lived **Kubernetes ServiceAccount token** (a JWT). Mint one in your namespace:

```bash
# see what you're allowed to do, and whether a service account already exists
kubectl auth can-i create serviceaccounts/token -n <your-namespace>
kubectl get serviceaccounts -n <your-namespace>

# if a service account already exists (e.g. one set up for your group), just mint a token for it:
kubectl create token <service-account> -n <your-namespace> --duration=720h
```

If you are a **namespace admin** and need to create one from scratch:

```bash
kubectl create serviceaccount faasr-runner -n <your-namespace>
kubectl create role faasr-jobs -n <your-namespace> \
  --verb=create,get,list,watch,delete --resource=jobs,pods,pods/log
kubectl create rolebinding faasr-runner-jobs -n <your-namespace> \
  --role=faasr-jobs --serviceaccount=<your-namespace>:faasr-runner
kubectl create token faasr-runner -n <your-namespace> --duration=720h
```

The command prints an `eyJ...` string — that is your token. (NRP may cap the duration; `--duration` requests an upper bound.)

## 6. Store your secrets in the FaaSr-workflow repo

In your [FaaSr-workflow repo], add GitHub Secrets (see [Creating cloud credentials]):

- **`<K8sServerName>_Token`** — the JWT from step 5. The name is your Kubernetes compute server's name followed by `_Token` (e.g. server `K8s` → secret `K8s_Token`).
- Your **data-store credentials** (e.g. `<DataStore>_AccessKey` / `<DataStore>_SecretKey`).

You can set the token from the CLI without printing it:

```bash
kubectl create token <service-account> -n <your-namespace> --duration=720h > k8s_token.txt
gh secret set K8s_Token --repo <your-username>/FaaSr-workflow < k8s_token.txt
```

## 7. Get the endpoint and CA certificate for the config

```bash
# API server endpoint
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'; echo

# CA certificate (base64) — this is the SSLCertificate value
kubectl config view --minify --raw \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'; echo
```

## 8. Build the workflow configuration

Configure two compute servers: your normal **GitHub Actions** server (which submits the Jobs) and the **Kubernetes** server. See [Running on Kubernetes] for the full field reference; the NRP-specific essentials:

```json
"ComputeServers": {
  "GH": {
    "FaaSType": "GitHubActions",
    "UserName": "YOUR_USERNAME",
    "ActionRepoName": "FaaSr-workflow",
    "Branch": "main",
    "UseSecretStore": true
  },
  "K8s": {
    "FaaSType": "Kubernetes",
    "Endpoint": "https://<nrp-api-server>",
    "Namespace": "<your-namespace>",
    "UseSecretStore": false,
    "AllowSelfSignedCertificate": true,
    "SSLCertificate": "<base64 CA cert from step 7>",
    "MaxCPU": 500,
    "MaxMemory": 1000,
    "TimeLimit": 300,
    "AdditionalTimeToLive": 60,
    "NumberOfRetries": 10
  }
}
```

!!! important "`UseSecretStore` must be `false` on the Kubernetes server"
    This is how your data-store credentials get passed to the Job's pod. With `UseSecretStore: true`, the pod receives no S3 credentials and fails with `NoCredentialsError`. The GitHub Actions server keeps `UseSecretStore: true`; the cluster token still comes from the `<K8sServerName>_Token` secret.

Point the Kubernetes action(s) at a FaaSr Kubernetes container image under `ActionContainers` (e.g. `faasr/kubernetes-r` / `faasr/kubernetes-python`, or your own image that the cluster can pull).

## 9. Register and invoke

1. Upload your workflow JSON to your [FaaSr-workflow repo].
2. Run **`FAASR REGISTER`** (tick *Allow custom containers* if your image is not a native FaaSr image) — see [Registering workflows].
3. Run **`FAASR INVOKE`** — see [Invoking workflows]. The entry action runs on GitHub Actions and submits a **Kubernetes Job per Kubernetes action** into your namespace.

## 10. Watch and inspect the Job

```bash
kubectl get jobs,pods -n <your-namespace> -w      # watch the Job appear and complete
kubectl logs <pod> -n <your-namespace>            # the function's output (S3 reads/writes)
kubectl describe job <job> -n <your-namespace>    # events, completions, deadline
kubectl get job <job> -n <your-namespace> -o yaml # full spec: image, resources, TTL
```

A finished Job (and its pod, and thus its logs) is auto-deleted after `AdditionalTimeToLive` seconds (`ttlSecondsAfterFinished`). **Bump `AdditionalTimeToLive`** in the config if you want a completed Job to linger for inspection.

## 11. A real-world example: the FLARE lake forecast

The tutorial workflow above is deliberately tiny. [FLARE](https://github.com/FLARE-forecast/FLAREr) (Forecasting Lake And Reservoir Ecosystems) is a real GLM-AED data-assimilation forecast — far heavier — which makes it a good end-to-end test of a Kubernetes action that does substantial compute and S3 I/O.

**Shape of the workflow** — one GitHub Actions action feeding one Kubernetes action:

```
start (GitHub Actions)  ──▶  run-fcre-aed-forecast (Kubernetes)
```

- `start` runs on GitHub Actions and seeds a trivial input (it satisfies FaaSr's "entry action must be GitHub Actions with `UseSecretStore: true`" rule).
- `run-fcre-aed-forecast` runs on the cluster and does the actual forecasting.

**The container.** The Kubernetes action uses a purpose-built image that bakes in GLM+AED (built from source), the **FLAREr** R package (installed from the public [`FLARE-forecast/FLAREr`](https://github.com/FLARE-forecast/FLAREr)), and a Kubernetes-capable FaaSr engine. The forecast entry function (`run_fcre_aed_forecast`) is pulled at run time from the FCRE forecast repository via `FunctionGitRepo`.

**Size the resources — FLARE is much heavier than the tutorial:**

```json
"K8s": {
  "FaaSType": "Kubernetes",
  "Endpoint": "https://<nrp-api-server>",
  "Namespace": "<your-namespace>",
  "UseSecretStore": false,
  "AllowSelfSignedCertificate": true,
  "SSLCertificate": "<base64 CA cert>",
  "MaxCPU": 2000,
  "MaxMemory": 8000,
  "TimeLimit": 1800,
  "AdditionalTimeToLive": 3600,
  "NumberOfRetries": 1
}
```

The forecast also needs its **object-store data stores** (met/inflow drivers, targets, forecast/score outputs, restart) configured with `UseSecretStore: false` so their credentials are passed into the pod — exactly as in step 8. **Register and invoke** as in steps 9–10; the GitHub Actions entry action submits one Kubernetes Job for `run-fcre-aed-forecast` into your namespace.

**What happens on the cluster.** Each forecast cycle assimilates the latest observations, runs a GLM-AED ensemble forecast for one reference date, writes the forecast and its skill scores to S3, then advances one day and writes a restart. The action loops over reference dates until its internal time budget, then exits cleanly. In a bounded demo run, the Job reached `Complete 1/1` in ~12 minutes, producing **6 daily forecasts** (`reference_date=2026-05-24 … 2026-05-29`) written to the object store, and exited gracefully at its time budget (`Approaching action time budget; exiting loop cleanly with restart written`).

!!! note "Long forecasts and `TimeLimit`"
    A full multi-month backfill can run for hours. Kubernetes kills a Job at `TimeLimit` (`activeDeadlineSeconds`) with `DeadlineExceeded`. For a **bounded** run, cap the forecast's internal loop comfortably below `TimeLimit` so it exits gracefully with a restart written; for the **full** range, raise `TimeLimit`. Because each cycle persists a restart to S3, a run that stops early can be resumed simply by invoking again.

!!! tip "The GLM namelist must match the GLM binary"
    A current GLM build expects the submerged-inflow variable `subm_height` in `glm3.nml`; older FCRE configs using `subm_elev` need that variable renamed, or GLM aborts with `Base nml missing the following variable name: subm_height`.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `NoCredentialsError` in the pod | The Kubernetes server has `UseSecretStore: true`; set it to **`false`**. |
| Browser login denied ("authenticate with the source you've previously signed up with") | Multi-account cookie collision — log in via a **fresh incognito** window (step 4). |
| Auth error on register/invoke | The service-account token expired — re-mint (step 5) and update the `<K8sServerName>_Token` secret. |
| Job shows `DeadlineExceeded` | The action ran longer than `TimeLimit` (or is stuck, e.g. retrying a failed S3 call). Raise `TimeLimit` or fix the underlying error. |
| `ImagePullBackOff` on the pod | The cluster can't pull your action image — make it public or push it somewhere Nautilus can reach. |
| GLM Job fails: `Base nml missing the following variable name: subm_height` | The GLM binary expects the current namelist schema; rename `subm_elev` → `subm_height` in your `glm3.nml`. |

[Running on Kubernetes]: kubernetes.md
[FaaSr-workflow repo]: workflow_repo.md
[Creating cloud credentials]: credentials.md
[Registering workflows]: register_workflow.md
[Invoking workflows]: invoke_workflow.md
