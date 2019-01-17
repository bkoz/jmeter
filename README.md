# Testing jmeter on OpenShift
## OpenShift v3.11

### Notes

Subscribe OpenShift nodes to RHT.

```
oc new-project jmeter
oc new-app https://github.com/bkoz/jmeter.git
```

A build configuration based on the Jenkinsfile should get created.

```
oc get bc
```

Login to the Jenkins console. The build should be running.

Looks like RHDPS imposed limits on new projects. Login as cluster admin and remove them.

```
oc delete limits jmeter-core-resource-limits
```

#### References
[1]https://eleanordare.com/blog/2018/5/3/using-a-jenkins-pipeline-to-run-jmeter-tests-in-openshift-bkba6
