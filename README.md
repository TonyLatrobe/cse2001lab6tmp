# gitJenkinTerraform

cse2001lab6: Jenkins, Terraform and OPA pipeline on MicroK8s

This lab runs a Jenkins pipeline inside Kubernetes (MicroK8s). Each stage runs in a pod that Jenkins starts for you.

Pipeline flow:

Debug → Unit Tests (pytest) → Build → Terraform Validate → OPA Policy Check → Terraform Security (Checkov) → Deploy

## Prerequisites

The lab VM from Week 1. It already has MicroK8s, the local registry (localhost:32000) and every image this lab uses (Jenkins, the Jenkins agent and python-ci). Nothing is downloaded during the lab except your own repository from GitHub.

Everything runs in its own namespace, `devopslab6`, so it never overlaps another lab.

## Project structure

```
cse2001lab6/
├── Jenkinsfile
├── Dockerfile.test
├── README.md
├── app/
│   ├── opa/terraform.rego
│   ├── requirements.txt
│   ├── src/add.py
│   ├── src/app.py
│   └── tests/test_app.py
├── jenkins/pod-templates/
│   ├── devops.yaml            (used by the Jenkinsfile)
│   ├── deploy.yaml  python.yaml  security.yaml  terraform.yaml   (used in a later lab)
├── k8s/
│   ├── namespace.yaml  jenkins-sa.yaml  jenkins-rbac.yaml  jenkins-pvc.yaml
│   ├── jenkins-ca-configmap.yaml  jenkins-deployment.yaml  jenkins-service.yaml
└── terraform/
    ├── main.tf
    └── provider.tf
```

## 1. Apply the Kubernetes manifests (in this order)

The namespace must exist first. Service account, RBAC and storage must be ready before the Jenkins Deployment.

```bash
sudo microk8s kubectl apply -f k8s/namespace.yaml
sudo microk8s kubectl apply -f k8s/jenkins-sa.yaml
sudo microk8s kubectl apply -f k8s/jenkins-rbac.yaml
sudo microk8s kubectl apply -f k8s/jenkins-pvc.yaml
sudo microk8s kubectl apply -f k8s/jenkins-ca-configmap.yaml
sudo microk8s kubectl apply -f k8s/jenkins-deployment.yaml
sudo microk8s kubectl apply -f k8s/jenkins-service.yaml
```

Check that everything starts (repeat until the Jenkins pod is Running):

```bash
sudo microk8s kubectl get all -n devopslab6
sudo microk8s kubectl get pvc -n devopslab6      # STATUS must be Bound
```

## 2. Log in to Jenkins

```bash
sudo microk8s kubectl get pods -n devopslab6
sudo microk8s kubectl exec -it deploy/jenkins -n devopslab6 -- cat /var/jenkins_home/secrets/initialAdminPassword
```

Open http://127.0.0.1:32043 in the browser on your computer (bridged network: http://<VM_IP>:32043).

Username `admin`, password: the value printed above.

If Jenkins says it appears to be offline, click **Skip plugin installations**. The plugins (Kubernetes, Git, GitHub) are already installed.

## 3. Configure Jenkins

1. **Manage Jenkins → System**, set the Jenkins URL to `http://jenkins.devopslab6.svc.cluster.local:8080/`.
2. **Manage Jenkins → Clouds → New cloud → Kubernetes:**
   - Name: `KubernetesCloud`
   - Kubernetes URL: `https://kubernetes.default.svc`
   - Kubernetes Namespace: `devopslab6`
   - Jenkins URL: `http://jenkins.devopslab6.svc.cluster.local:8080/`
   - Credentials: leave blank (GitHub credentials are added in the pipeline, not here)
   - Jenkins tunnel: `jenkins.devopslab6.svc.cluster.local:50000`
   - Disable https certificate check: tick
3. Under the same cloud, **Pod Template:**
   - Name: `pipelinepod`
   - Container name: `jnlp`
   - Docker image: `localhost:32000/jenkins-inbound-agent:lab`
   - Working directory: `/home/jenkins/agent`
   - Leave Command and Arguments empty
   - Pod service account: `jenkins`

All images come from the local registry (localhost:32000), so pods start in seconds.

## 4. Create and run the pipeline

1. **New Item** → name it `pipelinetest` → **Pipeline** → OK.
2. Definition: **Pipeline script from SCM**, SCM: **Git**.
3. Repository URL: `https://github.com/<username>/cse2001lab6`
4. Branch specifier: `*/main`, Script Path: `Jenkinsfile`, tick **Lightweight checkout**.
5. Save, then **Build Now**. Open **Console Output** to watch each stage.

### Expected results

- All stages green.
- Unit tests pass.
- Terraform validation succeeds.
- OPA policy check passes (no deny results).
- Checkov failure rate is at most 20% (the pipeline fails above that).
- Deploy prints `The sum of 3.0 and 5.0 is 8.0`.

## Cleaning up

Do this at the end of the lab to free memory for the next one:

```bash
sudo microk8s kubectl delete namespace devopslab6
```

## Troubleshooting

| Symptom | What to try |
|---|---|
| Image pull fails (ErrImagePull) | `curl http://127.0.0.1:32000/v2/_catalog` should list jenkins-secops, jenkins-inbound-agent and python-ci. `sudo microk8s kubectl describe pod <pod> -n devopslab6` shows which image failed. |
| Pipeline fails | `sudo microk8s kubectl get pods -n devopslab6`, then `sudo microk8s kubectl logs -c <container> <pod> -n devopslab6`. Also read the Jenkins Console Output. |
| Jenkins pod stuck Pending | `sudo microk8s kubectl get pvc -n devopslab6` must show Bound. |
| Jenkins will not load, or certificate errors (bridged network, VM IP changed) | Run `sudo startJenkinsLabDHCPcerts` (add `--force` if it still fails), then reload the page. |
| Still failing after that | `sudo microk8s refresh-certs --cert server.crt`, then `sudo microk8s stop` and `sudo microk8s start`. Clear your browser cache. |

Useful checks:

```bash
sudo microk8s kubectl get pods -n devopslab6
sudo microk8s kubectl get svc jenkins -n devopslab6
```
