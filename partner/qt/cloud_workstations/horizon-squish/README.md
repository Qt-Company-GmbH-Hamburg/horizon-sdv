# Squish for Horizon

A cloud workstation providing a ready to use Squish installation.

More information regarding Squish may be found at the [official documentation](https://doc.qt.io/squish/squish-for-android-tutorials.html).

## Initial setup

### Horizon SDV deployment guide

A working Horizon SDV environment is a basic requirement.

Follow [Horizon SDV deployment guide](https://github.com/GoogleCloudPlatform/horizon-sdv/blob/main/docs/deployment_guide.md)
and check notes below referring its subsections.

### ad. #2c Configure Terraform Variables

Follow the guide with all variables but a few settings may need customization:
- `sdv_enable_network_policies`: `false` - prevents network issues
- `sdv_dns_dnssec_enabled`: `false` - simplifies DNS handling

### ad. #3e Jenkins Access

At the moment of writing it seems that the guide omits an important part of granting per-user
Jenkins admin access. Without it **Seed Workloads** job breaks with an error.

Running the Seed Workloads and image-build jobs requires Jenkins **administrator** rights,
and adding a user to `administrators` group in KeyCloak is not enough (probably due to a bug).
Please remember to additionally assign the global `administrators` role
in [`values-jenkins.yaml`](../../../../gitops/workloads/values-jenkins.yaml) - this is the
**permanent** solution, because setting this directly in the Jenkins UI is reverted the next
time the configuration reloads.

Under `controller:JCasC:configScripts:...:roles:global:` find the `administrators` role and add a
`- user:` entry (uncomment the example and set your email):
```yaml
- name: "administrators"
  description: "Full Jenkins administration (Overall/Administer)."
  permissions:
    - "Overall/Administer"
  entries:
    - group: "administrators"
    - user: "you@example.com"
```

Then commit, push (Argo CD syncs it) and **re-login** to Jenkins (roles are evaluated at
login, so the new access only appears in a fresh session).

A full pod recreate may be needed for JCasC to pick up the change (a container restart is not enough):

```bash
kubectl -n jenkins delete pod jenkins-0
```

Instead of editing by hand you can apply the patch from the repository root (set your own email in
it first):
```bash
git am partner/qt/cloud_workstations/horizon-squish/patches/0001-manual-grant-Jenkins-administrators-role.patch
```

## Horizon SDV adjustments

### Move files from Qt partner directory

Move the job definition into the Jenkins Seed-scanned pipeline tree:
```bash
mkdir -p workloads/cloud-workstations/pipelines/workstation-images/horizon-squish/groovy
git mv partner/qt/cloud_workstations/horizon-squish/groovy/job.groovy \
       workloads/cloud-workstations/pipelines/workstation-images/horizon-squish/groovy/job.groovy
```

Commit and push (Argo CD syncs), then re-run **Seed Workloads** (`SEED_WORKLOAD=cloud-workstations`)
so the job is generated.

Instead of the `git mv` above you can apply the patch from the repository root:
```bash
git am partner/qt/cloud_workstations/horizon-squish/patches/0002-manual-move-Squish-job.groovy-into-scanned-path.patch
```

> The `Dockerfile` and `assets/` intentionally stay under `partner/...` - the Jenkinsfile builds
> the image from that directory.

### Additions to the `values-jenkins.yaml`

Please provide the following additions to the [`values-jenkins.yaml`](../../../../gitops/workloads/values-jenkins.yaml):

- under `controller:JCasC:configScripts:welcome-message:globalNodeProperties:envVars:env`:
```yaml
- key: "CLOUD_WS_HORIZON_SQUISH_FOR_ANDROID_IMAGE_NAME"
  value: {{ .Values.config.workloads.cloudWorkstations.workstationPresets.wsImages.horizonSquishForAndroid.image }}
```

- under `config:workloads:cloudWorkstations:workstationPresets:wsImages`:
```yaml
horizonSquishForAndroid:
    image: "horizon-sdv/cloud-ws-images/horizon-squish"
```

Instead of editing by hand you can apply the patch from the repository root:
```bash
git am partner/qt/cloud_workstations/horizon-squish/patches/0003-manual-register-the-Squish-image.patch
```

### Additions to the `Jenkinsfile`

The Seed job substitutes `${VAR}` tokens in the pipeline `job.groovy` files before running Job
DSL, to avoid the sandbox-blocked `getProperty` method.
Without it, seeding the Squish job fails with `Scripts not permitted to use method
groovy.lang.GroovyObject getProperty ... CLOUD_WS_HORIZON_SQUISH_FOR_ANDROID_IMAGE_NAME`.

1. Add the Squish image-name variable to the cloud-workstations `replacements` array in
[`workloads/seed/Jenkinsfile`](../../../../workloads/seed/Jenkinsfile), next to the other
`CLOUD_WS_HORIZON_*_IMAGE_NAME` entries:
```groovy
['${CLOUD_WS_HORIZON_SQUISH_FOR_ANDROID_IMAGE_NAME}', "${CLOUD_WS_HORIZON_SQUISH_FOR_ANDROID_IMAGE_NAME}"],
```
   Instead of editing by hand you can apply the patch from the repository root (covers this step
   only):
   ```bash
   git am partner/qt/cloud_workstations/horizon-squish/patches/0004-manual-map-Squish-image-name-in-seed-replacements.patch
   ```

2. under [`Jenkinsfile`](../../../../workloads/android/pipelines/environment/mirror/sync_mirror/Jenkinsfile)
append to the array describing the `mainKubernetesPodTemplateHighSpec`,
under `spec:affinity:podAffinity:requiredDuringSchedulingIgnoredDuringExecution:`:
```yaml
- labelSelector:
    matchExpressions:
    - key: cloud_ws_horizon_squish_for_android_build_pod
      operator: Exists
  topologyKey: kubernetes.io/hostname
```

## Squish License Key and URL

Contact [The Qt Company](https://www.qt.io/quality-assurance/download) to provide you with:
- a Squish installer URL
- a valid Squish License Key (Docker scenario doesn't allow license server use)

### Provide the License Key

Terraform creates the Secret Manager secret `squish-license-key` (empty) and grants
`jenkins-sa` read access - but it does **not** store the key value, so the license key
never lands in `terraform.tfvars` or Terraform state.

After `terraform apply`, add the key value out-of-band (once):
```bash
printf '%s' '<LICENSE_KEY>' | \
  gcloud secrets versions add squish-license-key \
    --project="<GCP_PROJECT_ID>" --data-file=-
```

The key will be detected automatically and used when building the `Horizon Squish for Android`
workstation image.

### Add the URL to the Dockerfile

1. Open the [Dockerfile](Dockerfile)

2. Navigate to the `RUN` command

3. Replace `<URL>` with the one provided by The Qt Company

### Troubleshooting

If the build fails during the "Create Docker Image" stage, check the following:

- Expired License: Verify the license string in Secret Manager (`squish-license-key`) is still
  valid with The Qt Company. Read the currently-stored value with:
  ```bash
  gcloud secrets versions access latest --secret=squish-license-key --project="<GCP_PROJECT_ID>"
  ```

- URL still valid: Verify the installer URL is still valid with The Qt Company.

## Build and run the Squish workstation

After **Seed Workloads** has generated the pipelines, build the image and launch a workstation
from it. All *create* steps are Jenkins pipelines under **Cloud-Workstations**; the `gcloud`
commands below are only for verifying each step.

### 1. Build the image
1. **Environment > Docker Image Template** with `NO_PUSH=false` - builds the shared env image the
   Cloud Workstations pipelines run inside.
2. **Workstation-Images > Horizon Squish for Android** with `NO_PUSH=false` - builds and pushes
   the Squish image. Verify it was pushed:
   ```bash
   gcloud artifacts docker images list \
     <REGION>-docker.pkg.dev/<GCP_PROJECT_ID>/horizon-sdv/cloud-ws-images/horizon-squish \
     --include-tags --project="<GCP_PROJECT_ID>"
   ```

Note: if the build sits `Pending` and eventually times out, the build pod does not fit your
project's CPU quota. Apply the build-pod patch from the repository root - it shrinks the pod
(dropping the dedicated `android` node pool and requesting 2-3 vCPU instead of 98) so it schedules
on the default pool:
```bash
git am partner/qt/cloud_workstations/horizon-squish/patches/0005-manual-fit-Squish-image-build-pod-within-CPU-quota.patch
```

### 2. Create cluster > config > workstation

Each resource is the parent of the next, so create them in sequence.

1. **Cluster-Admin-Operations > Create Cluster**

   Then verify it becomes `ACTIVE`:

   ```bash
   gcloud workstations clusters list --region="<REGION>" --project="<GCP_PROJECT_ID>"
   ```
2. **Config-Admin-Operations > Create New Configuration** with:
   - `CLOUD_WS_CONFIG_NAME` - e.g. `squish-config`
   - `HOST_DISABLE_SSH` = `false` (uncheck it)
   - `CONTAINER_IMAGE` = `<REGION>-docker.pkg.dev/<GCP_PROJECT_ID>/horizon-sdv/cloud-ws-images/horizon-squish:latest` (probably only the last part needs an update)
   - `WS_ALLOWED_PORTS` = `[{"first":22,"last":22},{"first":80,"last":80},{"first":1024,"last":65535}]`
   - `WS_ADMIN_IAM_MEMBERS` = `<YOUR_EMAIL>`

   Verify:

   ```bash
   gcloud workstations configs list --cluster="<CLUSTER_NAME>" \
     --region="<REGION>" --project="<GCP_PROJECT_ID>"
   ```

3. **Workstation-Admin-Operations > Create New Workstation** with:
   - `WORKSTATION_NAME` - e.g. `squish-ws`
   - `WORKSTATION_CONFIG_NAME` - e.g. `squish-config`
   - `INITIAL_WORKSTATION_USER_EMAILS_TO_ADD` = `<YOUR_EMAIL>`

   Then verify:
   ```bash
   gcloud workstations list --cluster="<CLUSTER_NAME>" --config="<CONFIG_NAME>" \
     --region="<REGION>" --project="<GCP_PROJECT_ID>"
   ```

### Start Workstation
Use **Workstation-User-Operations > Start Workstation** with:
   - `WORKSTATION_NAME` - e.g. `squish-ws`

Then open `Console Output` view for the job and wait for it to finish - near the end of the listing
you'll find a URL like:

```
2026-07-24 09:24:14.887   [SUCCESS] URL to access Workstation: https://80-squish-ws.cluster-zbyo6p3frnblctig74jnwpj6rg.cloudworkstations.dev
```

Follow the link to open a **noVNC** view in your browser.

> **First launch takes a few minutes.** This is a heavy image (GNOME + Android SDKs + Squish), and
> the desktop services start after boot. If noVNC shows *"Unable to forward your request to a
> backend / Couldn't connect to a server on port 80"* (or SSH reports no server on port 22) right
> after starting, the container is still coming up - wait a couple of minutes and retry before
> assuming something is broken.

In your browser you should see a virtual desktop via noVNC - to open Squish IDE:
- click the launcher button in the top left corner
- open "Show Apps" (rightmost dotted button)
- open "Squish for Android..."

At this point you should see a Squish IDE window.

Note: if Squish IDE complains about license key expiration you should:
- ask Qt for a new license key and send it to GCP (see "Provide the License Key" section)
- stop workstation (see section below)
- trigger **Workstation-Images > Horizon Squish for Android** again (see "Build the image" section)
- trigger **Workstation-Admin-Operations > Create New Workstation** again
- start workstation again

### Stop Workstation
After finished work you may stop the workstation via **Workstation-User-Operations > Stop Workstation** with:
   - `WORKSTATION_NAME` - e.g. `squish-ws`

### SSH access

If you need to debug the workstation image open a shell on a running workstation:
```bash
gcloud workstations ssh <WORKSTATION_NAME> \
  --project="<GCP_PROJECT_ID>" --region="<REGION>" \
  --cluster="<CLUSTER_NAME>" --config="<CONFIG_NAME>"
```
