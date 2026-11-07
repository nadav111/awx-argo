# Deploy Argo CD to k3s with Ansible and AWX

Your k3s cluster and AWX already exist. This repository is already connected to AWX,
and the test playbook ran successfully. The next step is to give the deployment
job the tools and Kubernetes credential it needs, then run the Argo CD playbook.

This project creates the `argocd` namespace and installs or upgrades Argo CD there.
It also connects the open source [podinfo](https://github.com/stefanprodan/podinfo)
demo as an Argo CD Application. It does not reinstall k3s or AWX.

## 1. How Ansible works

Ansible reads a **playbook**: a YAML file describing work to perform.
A playbook contains one or more **plays**. Each play selects hosts and lists
**tasks** to run. Each task calls a **module**, which performs a specific action.

For example, the `kubernetes.core.k8s` module can ensure a Kubernetes namespace
exists. You describe the result you want with `state: present`; the module checks
the existing state and creates the namespace when needed.

This is called **idempotence**: repeating the same instructions should leave the
same desired result rather than create duplicates. Modules and settings determine
whether a task behaves this way. See the [Ansible playbook introduction](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_intro.html).

For this repository, the flow is:

```text
Your Git repository
        |
        v
AWX Project syncs the files
        |
        v
AWX Job Template selects the playbook, inventory, credential and tools
        |
        v
Ansible runs the tasks inside the AWX job container
        |
        v
Helm and the Kubernetes client contact the k3s API
        |
        v
k3s runs Argo CD pods in the argocd namespace
```

**AWX** is the interface that starts and records the Ansible job.
**k3s** is the Kubernetes cluster where Argo CD runs.
**Helm** installs a packaged set of Kubernetes resources called a **chart**.
An installed chart is a **release**; our release is named `argocd`.
**Argo CD** watches the podinfo Git repository and deploys its Kubernetes manifests
into the `podinfo` namespace.

## 2. What each file does

```text
.
|-- README.md
|-- ansible.cfg                    # Local Ansible defaults
|-- inventory.yml                  # Where Ansible runs these tasks
|-- requirements.yml               # Required Ansible collection and version
|-- ee/
|   `-- execution-environment.yml   # Build instructions for the AWX job image
|-- playbooks/
|   |-- deploy-argocd.yml           # Main playbook: settings and task order
|   `-- tasks/
|       |-- check.yml              # Check that KUBECONFIG was supplied
|       |-- namespace.yml          # Ensure the namespace exists
|       |-- install.yml            # Register the chart repository and deploy
|       |-- ingress.yml            # Expose Argo CD over HTTP on port 80
|       `-- sample-app.yml         # Connect podinfo to Argo CD
`-- values/
    `-- argocd.yml                 # Settings passed to the Argo CD chart
```

The old `awx-test.yaml` has been removed. Use `playbooks/deploy-argocd.yml` in the
AWX Job Template from now on.

## 3. Understand 127.0.0.1 and the different users

The inventory contains:

```yaml
all:
  hosts:
    127.0.0.1:
      ansible_connection: local
      ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

- `all` is the group containing inventory hosts.
- `127.0.0.1` is the host selected by our playbook.
- `ansible_connection: local` runs modules in the job's own environment without SSH.
- `ansible_python_interpreter` uses the same Python that runs Ansible, so the
  modules can find the Python packages installed in that environment.

**Inside an AWX job, localhost is the execution container.** Even though AWX and
k3s are on the same physical server, the job does not automatically have access
to the server's files or its localhost network.

There are three separate identities:

| Identity | What it controls |
| --- | --- |
| Your AWX user | Whether you can configure and launch AWX jobs |
| Kubernetes identity in the kubeconfig | Whether the job may create resources in k3s |
| Your Argo CD user | Whether you can sign in to Argo CD and manage applications |

Setting `ansible_user` would select an SSH user for an SSH connection. This
playbook uses a local connection and Kubernetes API authentication, so it needs
no Machine/SSH credential or `ansible_user`.

## 4. Read the main playbook

File: [playbooks/deploy-argocd.yml](playbooks/deploy-argocd.yml).

```yaml
- name: Deploy Argo CD to k3s
  hosts: "127.0.0.1"
  connection: local
  gather_facts: false
  become: false
```

| Line | Meaning |
| --- | --- |
| `name` | A readable label displayed in the AWX job output |
| `hosts` | Select the matching inventory host |
| `connection: local` | Run locally inside the execution environment |
| `gather_facts: false` | Skip collecting OS details because these tasks do not use them |
| `become: false` | Do not request sudo; authorization comes from the Kubernetes credential |

The settings are:

```yaml
vars:
  argocd_namespace: argocd
  argocd_chart_version: "10.14.0"
  kubeconfig: "{{ lookup('ansible.builtin.env', 'KUBECONFIG') }}"
```

- `vars` defines values that tasks can reuse.
- `argocd_namespace` is the Kubernetes namespace for this installation.
- `argocd_chart_version` pins the Helm chart version so later launches do not
  silently select a newer chart. Chart versions and Argo CD application versions
  are separate version numbers.
- `lookup(..., 'KUBECONFIG')` reads an environment variable containing the path to
  a kubeconfig file. It reads the path, not the file's contents.
- `{{ ... }}` is Ansible's template syntax: evaluate the expression and use its value.

The `tasks` section imports the five task files in order:

```yaml
- name: Check cluster credential
  ansible.builtin.import_tasks: tasks/check.yml
```

`import_tasks` puts the tasks from that file into the play. It keeps the main
playbook short while each operation remains easy to read. Imported tasks use
the same hosts and variables as the play.

YAML uses indentation to show structure. Use spaces, not tabs. A leading `-`
starts an item in a list; `key: value` defines a setting.

## 5. Understand the task files

### Check the credential

File: [playbooks/tasks/check.yml](playbooks/tasks/check.yml).

```yaml
ansible.builtin.assert:
  that:
    - kubeconfig | length > 0
```

`assert` stops the job with an explanation if a condition is false.
The `| length` filter counts characters. Here it checks that the kubeconfig path
is not empty. It does not prove the file is valid or the account has permission;
the Kubernetes task checks those things when it contacts the API.

### Create the namespace

File: [playbooks/tasks/namespace.yml](playbooks/tasks/namespace.yml).

```yaml
kubernetes.core.k8s:
  kubeconfig: "{{ kubeconfig }}"
  validate_certs: true
  state: present
  definition:
    apiVersion: v1
    kind: Namespace
    metadata:
      name: "{{ argocd_namespace }}"
```

- `kubernetes.core.k8s` is a module from the `kubernetes.core` collection.
- `kubeconfig` tells the module which cluster and identity to use.
- `validate_certs: true` verifies the Kubernetes API server's TLS certificate.
- `state: present` means the resource should exist. An existing matching
  namespace is reused.
- `definition` contains the Kubernetes resource description.
- `kind: Namespace` identifies the resource type; `metadata.name` names it.

### Install Argo CD

File: [playbooks/tasks/install.yml](playbooks/tasks/install.yml).

The first task registers the upstream chart repository under the local name
`argo`. This is a **Helm chart repository**, separate from your Git repository.
The second task installs or upgrades its `argo-cd` chart.

| Setting | Meaning in this project |
| --- | --- |
| `release_name: argocd` | Reuse the same Helm installation on subsequent runs |
| `release_namespace` | Put the release in the configured namespace |
| `chart_ref: argo/argo-cd` | Select `argo-cd` from the repository named `argo` |
| `chart_version` | Use the version pinned in the playbook |
| `release_state: present` | Ensure the release is installed |
| `values_files` | Read our chart settings from `values/argocd.yml` |
| `update_repo_cache: true` | Refresh Helm's list of available charts |
| `reset_values: true` | Apply chart defaults and our file instead of retaining old custom overrides |
| `wait: true` | Wait for Helm's Kubernetes readiness checks |
| `timeout: 10m0s` | Allow up to ten minutes for the Helm operation |
| `atomic: true` | Remove a failed new installation or attempt rollback of a failed upgrade |
| `history_max: 10` | Retain up to ten Helm release revisions |
| `no_log: true` | Hide this task's output because Helm results can include Secrets |
| `diff: false` | Do not display this task's resource differences |

`playbook_dir` is the main playbook's directory. The path
`{{ playbook_dir }}/../values/argocd.yml` goes up one directory and into `values`.

Readiness checks are not a complete application test. Rollback also does not undo
all data or Kubernetes CustomResourceDefinition changes.
See the [Helm module reference](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/helm_module.html).

## 6. Understand the Argo CD settings

File: [values/argocd.yml](values/argocd.yml).

| Setting | Effect |
| --- | --- |
| `server.service.type: ClusterIP` | Keep the backend service inside the cluster |
| `server.ingress.enabled: false` | Let the Ansible task manage the Ingress separately from Helm |
| `configs.params.server.insecure: true` | Serve HTTP for ingress access on port 80 |
| `configs.cm.exec.enabled: "false"` | Disable Argo CD's browser terminal feature |
| `configs.rbac.policy.default: ""` | Grant no extra default role to authenticated users; assign permissions explicitly |

The bootstrap `admin` account still has administrator access. Additional users
need accounts or SSO and appropriate RBAC rules. RBAC means role-based access control.
Other settings use the pinned chart's defaults.

## 7. Prepare the execution environment

An **execution environment (EE)** is the container image containing the tools an
AWX job runs. Your test proved that AWX can pull the repository and run a simple
task. The deployment also needs Helm, the Kubernetes collection and Python client.

If your current EE already has these dependencies, use it. Otherwise, the build
definition is [ee/execution-environment.yml](ee/execution-environment.yml).

Its important sections are:

- `images.base_image`: the Linux image used as the starting point.
- `python_interpreter`: the Python version used inside the image.
- `ansible_core` and `ansible_runner`: the Ansible engine and AWX job runner.
- `galaxy: ../requirements.yml`: install the collection version listed in this repository.
- `python` and `system`: install Python libraries and Linux tools the tasks need.
- `additional_build_steps`: install a pinned Helm binary and verify its checksum.

The current Helm archive is for **Linux/amd64**. An ARM server needs a matching
archive and checksum before using this definition.

On a Linux/amd64 machine with Docker, from the repository root:

```sh
python3 -m venv .venv
. .venv/bin/activate
pip install ansible-builder==3.1.0

ansible-builder build -f ee/execution-environment.yml \
  -t YOUR_REGISTRY/awx-argocd-ee:1 --container-runtime docker

docker login YOUR_REGISTRY
docker push YOUR_REGISTRY/awx-argocd-ee:1
```

Replace `YOUR_REGISTRY` with your registry address. The builder and AWX must be
able to reach it. Register the pushed image under AWX **Execution Environments**;
attach registry credentials there if the image is private.

Building the image on the server does not automatically make it available to
k3s. Pushing it to a reachable registry lets AWX pull it for the job.
For an offline transfer, export the image with `docker save`, copy the archive
to the server, and import it with `k3s ctr -n k8s.io images import ARCHIVE`.
For AWX jobs running as Kubernetes pods, select the imported image and set its
pull policy to **Never pull container before running**. Import onto every node
where the job can run. Separate execution nodes need the image in their own runtime.
The base image tag and transitive dependencies can change between builds; use
an approved image digest when you need reproducible production builds.

## 8. Give the job a kubeconfig

A **kubeconfig** contains the cluster address, certificate information and an
identity used to access Kubernetes. The file's selected context determines the
target cluster. This project uses that selection; verify it before launching.

The k3s administrator kubeconfig is `/etc/rancher/k3s/k3s.yaml` on the server.
An authorized administrator can read it in a private terminal:

```sh
sudo cat /etc/rancher/k3s/k3s.yaml
```

That file grants unrestricted cluster administration. A dedicated deployment
identity with the permissions needed by this chart is preferable for ongoing
use. It needs namespace permissions and permission to manage the chart's
cluster-scoped CRDs, ClusterRoles and ClusterRoleBindings.
CRDs add Argo CD's resource types to Kubernetes; roles and bindings grant access.

For an initial lab deployment, you can use an authorized copy of the k3s admin
kubeconfig as the AWX credential. Keep it private and never commit it to Git.

In the **copy supplied to AWX**, change:

```yaml
server: https://127.0.0.1:6443
```

to an address reachable from the job container, for example:

```yaml
server: https://192.168.1.50:6443
```

Use your actual k3s server address. It must also match the API certificate;
configure k3s `tls-san` through your cluster administrator if needed. Do not turn
off TLS validation. Leave the embedded certificate data intact. See
[k3s cluster access](https://docs.k3s.io/cluster-access).

Keep the inventory host as `127.0.0.1`: that selects where Ansible executes.
The kubeconfig's `server` selects the Kubernetes API it contacts.

### Store it in AWX

An AWX administrator creates a custom credential type named **Kubeconfig**.
Paste this into **Input Configuration**:

```yaml
fields:
  - id: kube_config
    type: string
    label: Kubeconfig
    secret: true
    multiline: true
required:
  - kube_config
```

Paste this into **Injector Configuration**:

```yaml
file:
  template.kubeconfig: '{{ kube_config }}'
env:
  KUBECONFIG: '{{ tower.filename.kubeconfig }}'
```

The input is a secret text field. The injector writes that text to a temporary
file and sets `KUBECONFIG` to its path for the job.

Create a credential of this type named **k3s-deployment**. Paste the prepared
kubeconfig into its field and save it. Attach this credential to the deployment
Job Template. Do not paste the kubeconfig into Extra Variables.
See [AWX custom credential types](https://docs.ansible.com/projects/awx/en/24.6.1/userguide/credential_types.html).

If the kubeconfig uses external certificate files or an authentication helper,
those must also exist inside the job. The standard k3s file normally embeds its
certificates. Copied admin credentials must be refreshed when their certificates
expire; restarting k3s does not update the copy stored in AWX.

## 9. Configure the existing AWX project and launch

You already have a working project connection. Keep that project.

1. Commit and push these files to the branch your existing AWX Project uses.
2. Sync that Project and wait for the sync to succeed.
3. Use an inventory containing host **127.0.0.1**. You can import `inventory.yml`
   through a Project inventory source, or create the host manually and paste:

   ```yaml
   ansible_connection: local
   ansible_python_interpreter: "{{ ansible_playbook_python }}"
   ```

4. Create or edit a Job Template with these settings:

   | Field | Value |
   | --- | --- |
   | Job Type | Run |
   | Inventory | The inventory containing `127.0.0.1` |
   | Project | Your existing connected project |
   | Playbook | `playbooks/deploy-argocd.yml` |
   | Execution Environment | The image with the required dependencies |
   | Credentials | `k3s-deployment` |
   | Limit | Leave empty, or enter `127.0.0.1` |
   | Privilege Escalation | Disabled |
   | Concurrent Jobs | Disabled for this template |
   | Extra Variables | No overrides needed |

5. Save and launch the template. Choose **Run**, not **Check**, for installation.

The job needs network access to the k3s API and chart repository. The k3s nodes
need access to the container registries used by the chart.

## 10. Read the job output and verify

AWX displays each task's `name` and its result:

| Result | Meaning |
| --- | --- |
| `ok` | Task succeeded without reporting a change |
| `changed` | Task succeeded and reported a change |
| `failed` | Task failed; subsequent tasks on that host normally stop |
| `skipped` | Task did not run |

The Helm task's details are intentionally hidden by `no_log`. Its success or
failure still appears. Do not disable secret protection just to print a password.

On a machine with `kubectl` configured for this cluster:

```sh
kubectl -n argocd get pods
kubectl -n argocd get svc argocd-server
kubectl -n argocd get ingress argocd-server
```

`-n argocd` selects the namespace. Pods should become ready; the service should
have type `ClusterIP`. On the k3s server, `sudo k3s kubectl` can be used in place
of `kubectl` if your Linux user has no kubeconfig configured.

## 11. Open Argo CD as a user

After Helm installs Argo CD, the playbook runs
[playbooks/tasks/ingress.yml](playbooks/tasks/ingress.yml) automatically. This
idempotent task creates a Kubernetes Ingress for Traefik, which k3s installs by
default. It uses the `traefik` ingress class and `web` entrypoint on port 80,
forwarding `/` to `argocd-server:80` without a hostname restriction.
See [k3s networking services](https://docs.k3s.io/networking/networking-services).

Sync the AWX Project and run the deployment job, then open
**http://YOUR_K3S_NODE_IP** using a node IP reachable from your workstation where
Traefik is exposed. No DNS entry or TLS Secret is required. Port 80 must be
reachable, and Traefik must not globally redirect HTTP to HTTPS. HTTP traffic,
including login credentials, is unencrypted.

For CLI access through this Ingress, use:

```sh
argocd login YOUR_K3S_NODE_IP:80 --plaintext --grpc-web
```

As an alternative, on **your workstation** with a working kubeconfig:

```sh
kubectl -n argocd port-forward --address 127.0.0.1 svc/argocd-server 8080:80
```

Keep that terminal open. Visit **http://127.0.0.1:8080** in your workstation's
browser. This forwards local port `8080` to service port `80` inside k3s.
It binds only to your workstation's loopback address.

If you run port-forward on the server instead, that localhost address belongs
to the server. It does not automatically open port 8080 on your laptop.

In a second private terminal, retrieve the initial password:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 --decode
```

The username is **admin**. The command extracts and decodes the password; do not
run it in a shared AWX log. It assumes a shell with `base64`, such as Linux or WSL.
Change the password after signing in. After confirming the new password works,
delete the initial password Secret:

```sh
kubectl -n argocd delete secret argocd-initial-admin-secret
```

Deleting that Secret does not rotate the password by itself.
An AWX account does not become an Argo CD account. Use Argo CD accounts or SSO
and explicit permissions when adding more users.
See [Argo CD getting started](https://argo-cd.readthedocs.io/en/stable/getting_started/).

### Try the open source podinfo application

The final installation task creates an Argo CD `Application` named `podinfo`.
Ansible creates the Application; Argo CD fetches the public
[podinfo repository](https://github.com/stefanprodan/podinfo), renders its
`kustomize` directory and deploys it into the same cluster, in a separate
`podinfo` namespace. No Git credentials are required for this public repository.
The cluster needs access to GitHub and the container registry `ghcr.io`.

After syncing the AWX Project and launching the installation job:

1. Open Argo CD at `http://YOUR_K3S_NODE_IP` and select **podinfo**.
2. Watch it become **Synced** and **Healthy**. The resource tree shows the
   Deployment, Service, autoscaler and running pods. The AWX task creates the
   Application; its success does not mean podinfo is healthy yet.
3. Open the demo from your workstation:

   ```sh
   kubectl -n podinfo port-forward svc/podinfo 9898:9898
   ```

   Keep the command running and visit **http://127.0.0.1:9898**.

Automatic sync follows the upstream `master` branch. `selfHeal: true` corrects
manual changes in the cluster, and `prune: true` removes resources deleted from
the source manifests. These are
[Argo CD Application settings](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/).
To see self-healing, change the demo's UI color directly in Kubernetes:

```sh
kubectl -n podinfo set env deployment/podinfo PODINFO_UI_COLOR='#ff9900'
```

Argo CD detects the difference and restores the color defined in Git. In the UI,
use **Refresh** and inspect the resource diff; the difference may disappear
quickly as self-healing runs. For a lasting change, fork podinfo, update the
Application's `repoURL` in `playbooks/tasks/sample-app.yml`, and commit your
changes to that fork. Argo CD will then sync your Git changes automatically.

To skip creating or updating the sample Application on future AWX runs, set
`argocd_sample_app_enabled: false` in Extra Variables. This does not remove an
already deployed Application or its resources.

## 12. Change settings and rerun

- Change the chart version or namespace in `playbooks/deploy-argocd.yml`.
- Change Argo CD chart settings in `values/argocd.yml`.
- Commit, push, sync the AWX Project and launch the same template again.

The same release name and namespace are reused, so rerunning does not create a
second Argo CD installation. A run with no desired changes should report no
release change; repository/cache maintenance may still report changes.
Changing the namespace creates a separate target and leaves the old installation
in place, so choose it before the first deployment.

Check upstream upgrade instructions before changing the chart version. Keep
customizations in Git: `reset_values: true` means old values supplied outside
this repository are not automatically preserved on upgrade.

## 13. Common problems

| Problem | What to check |
| --- | --- |
| `No hosts matched` | Inventory contains `127.0.0.1`; template Limit matches or is empty |
| Missing kubeconfig assertion | Attach the custom credential to this Job Template |
| `helm: command not found` | Select an EE containing Helm 3 |
| Cannot resolve `kubernetes.core` module | Install the pinned collection in the selected EE |
| Missing Kubernetes Python library | Use an EE containing the Kubernetes Python client |
| Connection refused at `127.0.0.1:6443` | Replace the loopback API address in the AWX kubeconfig copy |
| Certificate verification error | Check the CA and server address against the API certificate |
| `Forbidden` | The kubeconfig identity lacks the required Kubernetes permissions |
| Pod `Pending` | Check node capacity, scheduling and resource requests |
| `ImagePullBackOff` | Check image registry access and credentials from k3s |
| Helm timeout | Inspect pods and events; a failed new atomic installation may already have been removed |

Useful non-password diagnostic commands:

```sh
kubectl -n argocd get pods
kubectl -n argocd get events --sort-by=.lastTimestamp
helm status argocd -n argocd
```

Run them against the same cluster as the AWX job. `helm status` requires Helm and
a configured kubeconfig. For a failed pod, `kubectl -n argocd describe pod POD_NAME`
shows scheduling and startup events; replace `POD_NAME` with its actual name.

## 14. Supporting Ansible settings

[requirements.yml](requirements.yml) pins `kubernetes.core` to `6.6.0`.
A collection is a package of Ansible modules; declaring it does not install Helm
or the Kubernetes Python client. The EE definition installs those separately.

[ansible.cfg](ansible.cfg) sets these local defaults:

| Setting | Purpose |
| --- | --- |
| `inventory = ./inventory.yml` | Default inventory when running Ansible from this repository locally |
| `interpreter_python = auto_silent` | Allow automatic Python discovery without discovery warnings |
| `retry_files_enabled = False` | Avoid creating retry files |
| `display_args_to_stdout = False` | Avoid adding module arguments to task titles |
| `[diff] always = False` | Do not enable diff output globally |

AWX still uses the Inventory selected in the Job Template. Its inventory choice
is not replaced by the local default in `ansible.cfg`.
