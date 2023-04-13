# Java Application with CRDA CLI pipeline task !
A basic sample application using Java Spring Boot with customized stonesoup tekton pipeline to include the CRDA static code analysis task.

All the CRDA cli dependencies are built into the docker image - `quay.io/lrangine/crda-maven:11.0`. If you want to customize this docker image then please take look at the [source code here](https://github.com/lokeshrangineni/crda-images).

## Configuring authentication tokens for CRDA CLI. 

`CRDA` CLI needs to have the authentication and CRDA token. You can find more information about obtaining those [tokens here](https://github.com/redhat-actions/crda#4-set-up-authentication).

If you have already configured CRDA CLI on your local environment then you can get all the required tokens using the command `crda config get`.

Once you obtain above tokens create the secret so that your pipeline can use above tokens required by CRDA CLI.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: crda-scan-secrets
  namespace: lrangine-tenant
type: Opaque
stringData:
  AUTH_TOKEN: 9e7da12345fe123d8c10fa123e12345f
  CRDA_KEY: 12345678-bf9c-45ad-b236-a04a62312d9a
  HOST: https://f8a-analytics-2445582058137.production.gw.apicast.io
```


## CRDA Task definition for Pipeline.

`crda-scan` task is added to the default java pipeline provided by the Stone Soup.

Here is the task definition.

```yaml
    - name: crda-scan
      description: CRDA tool Scans for static code vulneribilities.
      taskSpec:
        results:
        - name: CLAIR_SCAN_RESULT
          description: CRDA scan result
        - name: HACBS_TEST_OUTPUT
          description: CRDA scan test output
        workspaces:
          - name: source
        steps:
          - name: run-redhat-crda-scan-step
            image: quay.io/lrangine/crda-maven:11.0
            workingDir: "$(workspaces.source.path)"
            env:
              - name: AUTH_TOKEN
                valueFrom:
                  secretKeyRef:
                    name: crda-scan-secrets
                    key: AUTH_TOKEN
              - name: CONSENT_TELEMETRY
                value: "false"
              - name: CRDA_KEY
                valueFrom:
                  secretKeyRef:
                    name: crda-scan-secrets
                    key: CRDA_KEY
              - name: HOST
                valueFrom:
                  secretKeyRef:
                    name: crda-scan-secrets
                    key: HOST
            script: |
              #!/usr/bin/env bash
              /crda.sh pom.xml crda_scan_output.json
              echo 'CRDA Scan is completed'
              echo 'Printing scan results'
              more crda_scan_output.json
              echo 'Formatting the Task Results'
              jq -rce \
              '{vulnerabilities:{
              critical: (.report."critical_vulnerabilities"),
              high: (.report."high_vulnerabilities"),
              medium: (.report."medium_vulnerabilities"),
              low: (.report."low_vulnerabilities")
              }}' crda_scan_output.json | tee $(results.CLAIR_SCAN_RESULT.path)

              NOTE="Task $(context.task.name) completed: Refer to Tekton task result CLAIR_SCAN_RESULT for vulnerabilities scanned by CRDA."
              HACBS_TEST_OUTPUT="SUCCESS $NOTE"
              echo "${HACBS_TEST_OUTPUT}" | tee $(results.HACBS_TEST_OUTPUT.path)
      runAfter:
        - clone-repository
      workspaces:
      - name: source
        workspace: workspace
```
`crda-scan` task definition is dependent on `clone-repository` task. `clone-repository` task will clone github source code to the workspace then 'crda-scan' task will perform the static code analysis using CRDA CLI tool. 
