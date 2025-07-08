---
title: "k6 Deployment on k8s"
date: 2025-05-17
description: "A humorous and detailed account of diagnosing and fixing a Linux kernel panic triggered by a full /boot partition. Follow the journey through LUKS challenges, partition management, and the tools like Ventoy and BorgBackup that saved the day on an Arch Linux system."
tags:
  - "kubernetes"
  - "job"
  - "k6"
  - "load testing"
categories:
  - "k8s"
  - "kubernetes"
  - "k6"
  - "load testing"
author: "Rodrigo Broggi" # Or your actual name/handle
toc: true # Set to false if you don't want a Table of Contents
draft: false # Set to true if it's not ready to be published
image: "panic-qr-code.png" # Path to an image for the card
---

# **k6 Deployment on k8s**

This document provides a step-by-step guide for deploying and managing k6 load tests on Kubernetes using kubectl. This manual approach offers fine-grained control and is ideal for understanding the interaction between the different resources involved.

## **Overview of Resources and Files**

This workflow involves three key components:

1. **your-test-script.js**: This is your local [k6](https://k6.io/) load test script, written in JavaScript. You will manage and edit this file directly.
2. **ConfigMap (k6-load-test-script)**: A Kubernetes ConfigMap that stores your k6 script. By storing the script in a ConfigMap, you decouple your test logic from the execution environment. The Job's pod will mount this ConfigMap as a file.
3. **Job (k6-load-test-job)**: A Kubernetes Job that runs the k6 test to completion. It defines the k6 container, mounts the ConfigMap containing the script, and configures all necessary environment variables for the test, including credentials and endpoint configurations.

## **Creating the ConfigMap from the k6 Script**

Instead of manually pasting your script into a YAML file, you can generate the ConfigMap directly from your local `.js` file. This prevents formatting errors and keeps your workflow clean.

Use the following [kubectl](https://kubernetes.io/docs/reference/kubectl/) command to generate the manifest and save it to a file:

```sh
kubectl create configmap k6-load-test-script \
  --from-file=your-test-script.js \
  --dry-run=client -o yaml > k6-configmap.yaml
```

* `--from-file`: Specifies the local script to be included. The filename (your-test-script.js) will become the key in the ConfigMap's data.
* `--dry-run=client -o yaml`: Generates the YAML output without applying it to the cluster.

You can now apply this manifest: `kubectl apply -f k6-configmap.yaml`

If you use [Kustomize][kustomize], you can also add the script to your `kustomization.yaml` under [configMapGenerator][kustomize-configmap-generator] to automate this step.

## **The Job Manifest Explained**

The Job is the resource that orchestrates the test run. Below is an example manifest with key sections explained.

```yaml
# k6-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: k6-load-test-job
spec:
  template:
    spec:
      containers:
        - name: k6
          image: grafana/k6:latest
          args:
            - "run"
            - "/scripts/your-test-script.js"
            - "--out"
            - "experimental-prometheus-rw"
          env:
            # a. Prometheus Remote Write Configuration
            - name: K6_PROMETHEUS_RW_SERVER_URL
              value: "http://your-prometheus-endpoint/api/v1/write"
            # b. Dynamic Test ID Configuration
            - name: K6_TESTID_PREFIX
              value: "mytest-label"
            - name: K6_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name

            # b. Example of secret injection using a custom resource
            - name: K6_SECRET
              valueFrom:
                secretKeyRef:
                  name: test-secret # Name of the Kubernetes Secret
                  key: password
      volumes:
        - name: k6-script-volume
          configMap:
            name: k6-load-test-script # Mounts the ConfigMap
      restartPolicy: Never
  backoffLimit: 4
```

### **Prometheus Remote Write Integration**

To send metrics to Prometheus, you configure k6 via the `--out` flag and environment variables:

* `--out experimental-prometheus-rw`: This argument in the args section tells k6 to activate the Prometheus remote write output.
* `K6_PROMETHEUS_RW_SERVER_URL`: This environment variable specifies the endpoint where your Prometheus-compatible service (like Alloy, Grafana Mimir, or Prometheus itself) is listening for remote write requests.

With those metrics you can power custom or out-of-the-box dashboards in Grafana, or use them for alerting and analysis.

![K6 dashboard][k6-dashboard-image]

### **Dynamic Test ID for Unique Runs**

To distinguish between different runs of the same test, a unique testid is generated for each Job execution. This is achieved by combining a static prefix with a dynamic identifier from the Job's pod.

* `K6_TESTID_PREFIX`: An environment variable you set to a fixed string (e.g., `mytest-label`) to identify the test campaign.
* `K6_POD_NAME`: This variable is populated using the Kubernetes **Downward API**. valueFrom.fieldRef injects the pod's metadata (in this case, its name) into an environment variable. Since each pod in a Job has a unique name (e.g., `k6-load-test-job-abcde`), this provides the dynamic part of our ID.
   Not necessarily we want to use a high-granularity identifier like the pod name as this is generally not a good practice for prometheus metrics.
* *k6 Script Logic**: The k6 script is responsible for reading these two environment variables and combining them to form the final testid tag.

### **Injecting Secrets**

Managing credentials directly in manifests is not secure. The standard and secure way to provide sensitive data like passwords or tokens to a pod is by using Kubernetes Secrets.

* **valueFrom.secretKeyRef**: This configuration tells the container to read the value for the `K6_SECRET` environment variable from a Kubernetes Secret resource named `test-secret`, specifically from the data field with the key `password`. This decouples your Job from the sensitive value itself.

### **Sample k6 Script**

```javascript
import { options } from 'k6';

const testIdPrefix = __ENV.K6_TESTID_PREFIX || 'local-test';
const podName = __ENV.K6_POD_NAME || null;
const secret = __ENV.K6_SECRET

// 3. Construct the final testid.
// If K6_POD_NAME is available, use it to create a unique ID.
// Otherwise, use a timestamp for local runs.
const testId = podName ? `${testIdPrefix}-${podName}` : `${testIdPrefix}-${Date.now()}`;


// --- Test Options ---
export const options = {
    tags: {
        // The dynamically generated testid is applied to all metrics.
        testid: testId,
    },
    scenarios: {
        my_scenario: {
            executor: 'ramping-vus',
            startVUs: 0,
            stages: [
                { duration: '10s', target: 5 },
                { duration: '10s', target: 0 },
            ],
            exec: 'default',
        },
    },
};

// --- Main VU function ---
export default function () {
    console.log(`Running test with testid: ${testId}`);
    // Your test logic (e.g., http.get, etc.) goes here.
}
```

## **Update and Re-deployment Workflow**

Since Kubernetes Jobs are designed to run to completion and are immutable, you cannot update a running or completed Job directly. The standard workflow is to **delete the old Job and create a new one**.

### **Workflow A: Updating from Local Manifests**

Use this approach when you have the original your-test-script.js and k6-job.yaml files.

1. **Modify the files**: Make changes to your local .js script or k6-job.yaml as needed.
2. **Update the ConfigMap**: If you changed the script, update the ConfigMap in the cluster.

```sh
# This command updates the ConfigMap with the new script content
kubectl create configmap k6-load-test-script \
  --from-file=your-test-script.js \
  --dry-run=client -o yaml | kubectl apply -f -
```

3. **Delete and Re-create the Job**: To launch a new run with the updated script or parameters, delete the old Job and apply the manifest again.  

```sh
kubectl delete job k6-load-test-job --ignore-not-found=true
kubectl apply -f k6-job.yaml
```

### **Workflow B: Updating from Live Cluster Resources**

Use this workflow if you don't have the original manifest files and need to patch the live resources.

1. **Fetch the Live Resources**: Get the current ConfigMap and Job definitions from the cluster and save them locally.  
  ```sh
  # Fetch the ConfigMap
  kubectl get configmap k6-load-test-script -o yaml > k6-configmap-from-cluster.yaml

  # Fetch the Job
  kubectl get job k6-load-test-job -o yaml > k6-job-from-cluster.yaml
  ```

2. **Clean and Patch the Manifests**:
* Open both fetched YAML files (`k6-configmap-from-cluster.yaml` and `k6-job-from-cluster.yaml`).
* **Crucially, you must remove cluster-managed metadata fields** before you can re-apply them. Delete fields like `metadata.resourceVersion`, `metadata.uid`, `metadata.creationTimestamp`, and the entire status block.
* Make your desired changes, such as editing the inlined script in the ConfigMap or updating an environment variable in the Job.
3. **Apply the Patched Resources**:
* Apply the updated ConfigMap:  
  ```sh
  kubectl apply -f k6-configmap-from-cluster.yaml
  ```
* Delete the old Job and apply your patched version:  
  ```sh
  kubectl delete job k6-load-test-job --ignore-not-found=true
  kubectl apply -f k6-job-from-cluster.yaml
  ```
  
[k6-dashboard-image]: k6-dashboard-image.png
[kustomize]: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
[kustomize-configmap-generator]: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#configmapgenerator