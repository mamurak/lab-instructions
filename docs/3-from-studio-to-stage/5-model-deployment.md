## Model Deployment via GitOps

We deployed our `jukebox` model in experiment environment manually, but for the higher environments we need to store the definitions in Git and deploy our models via Argo CD to get all the benefits that GitOps brings.

# Deploying Jukebox 

1. In your code server IDE, open `mlops-gitops/values.yaml` file and **swap** `enabled: false` to `enabled: true` as shown below for each of the app-of-pb definitions:

    <div class="highlight" style="background: #f7f7f7">
    <pre><code class="language-yaml">
        # Test App of Apps for Models
        - name: model-deployments
          enabled: true
          source_path: "."
          helm_values:
            - test/values.yaml
        # Staging App of Apps for Models
        - name: model-deployments
          enabled: true
          source_path: "."
          helm_values:
            - stage/values.yaml
    </code></pre></div>

2. Let's add `jukebox` model definition under `model-deployments/test/values.yaml` and `model-deployments/stage/values.yaml` files as follow. This will take a model deployment helm-chart from a generic helm chart repository and apply the additional configuration to it from the `values` section. 

    ```yaml
      # Jukebox
      jukebox:
        name: jukebox
        enabled: true
        source: https://<GIT_SERVER>/<USER_NAME>/mlops-helmcharts.git
        source_ref: main
        source_path: charts/model-deployment/simple
        values:
            name: jukebox
            version: latest
    ```
3. Let's get this deployed of course - it's not real unless its in git!

    ```bash#test
    cd /opt/app-root/src/mlops-gitops
    git add .
    git commit -m  "🐰 ADD - app-of-apps and jukebox to test 🐰"
    git push 
    ```

4. With the values enabled, and the first application listed in the test environment - let's tell Argo CD to start picking up changes to these environments. To do this, simply update the helm chart we installed at the beginning of the first exercise:

    ```bash#test
    cd /opt/app-root/src/mlops-gitops
    helm upgrade --install argoapps --namespace <USER_NAME>-mlops .
    ```

5. You should see the two Jukebox application, one for `test` and one for `stage` deployed in Argo CD. 


# 🚗 Modelcar and GitOpsifying the Deployment

TODO: add Modelcar and Kserve explanation there

The training pipeline generates a new version of the model, whether it is triggered by a code change or a monitoring alert. As a next step, we need to make sure we deploy the new version automagically (yes, of course if only the new version performs better - hang in there🤭) and we need to be able to provide traceability, immutability, reproducibility and so on.

If the pipeline is triggered by a git push event, the new model gets short git commit ID as the version. This helps us to track where this model comes from, why it is built. 

At the end of the day, we need to provide this version information to `InferenceService`. When the pipeline successfully creates the model artifacts, it pushes this version info to `mlops-gitops/model-deployment/` repository. Then Argo CD becomes aware that there is a new version and it updates `InferenceService` config on OpenShift, which triggers a rollout deployment. Simples 😅