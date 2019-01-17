print "----------------------------------------------------"
print "                 JMeter Testing"
print "----------------------------------------------------"

node('jenkins-agent'){

  workspace = pwd()   // Set the main workspace in your Jenkins agent

  authToken = ""      // Get your user auth token for OpenShift
  apiURL = ""         // URL for your OpenShift cluster API

  gitUser = ""        // Set your Git username
  gitPass = ""        // Set your Git password
  gitURL = ""         // Set the URL of your test suite repo
  gitName = ""        // Set the name of your test suite repo
  gitBranch = ""      // Set the branch of your test suite repo

  // Set location of OpenShift objects in workspace
  buildConfigPath = "${workspace}/${gitName}/build-config.yaml"
  imageStreamPath = "${workspace}/${gitName}/image-stream.yaml"
  jobTemplatePath = "${workspace}/${gitName}/job-template.yaml"

  project = ""    // Set the OpenShift project you're working in
  testSuiteName = "jmeter-test-suite"   // Name of the job/build/imagestream

  // Login to the OpenShift cluster
  sh """
      set +x
      oc login --insecure-skip-tls-verify=true --token=${authToken} ${apiURL}
  """

  // Checkout the test suite repo into your Jenkins agent workspace
  // You can also use the "checkout scm" Jenkins step here if you like
  int slashIdx = gitURL.indexOf("://")
  String urlWithCreds = gitURL.substring(0, slashIdx + 3) +
          "\"${gitUser}:${gitPass}\"@" + gitURL.substring(slashIdx + 3);

  sh """
    rm -rf ${workspace}/${gitName}
    git clone -b ${gitBranch} ${urlWithCreds} ${gitName}
    echo `pwd && ls -l`
  """

  // Create your ImageStream and BuildConfig in OpenShift
  // Then start the build for the test suite image
  sh """
    oc apply -f ${imageStreamPath} -n ${project}
    oc apply -f ${buildConfigPath} -n ${project}
    oc start-build ${testSuiteName} -n ${project} --follow
  """

  // Get latest JMeter image from project to assign to job template
  String imageURL = sh (
    script:"""
      oc get is/rhel-${testSuiteName} -n ${project} --output=jsonpath={.status.dockerImageRepository}
      """,
    returnStdout: true
  )

  // Get all JMX files from JMeter directory
  // Pipeline Utility Steps Plugin --> findFiles
  String files = findFiles(glob: '**/jmeter/*.jmx')

  // Split file names into testFileNames array
  testFileNames = files.split('\n')

  // For every test file, run job and get results back
  for (int i=0; i<testFileNames.size(); i++) {

    // Get file name without .jmx
    file = testFileNames[i]
    fileName = file.replaceAll('.jmx','')

    print "Running JMeter tests: ${fileName}"

    // Register a web hook to notify on test completion
    hook = registerWebhook()
    hookUrl = hook.getURL()

    // Delete existing test suite job for previous JMX file
    // Create new test suite job with current file
    // Pass in webhook URL, test file name, and image
    sh """
      oc delete job/${testSuiteName} -n ${project} --ignore-not-found=true
      
      oc process -f ${jobTemplatePath} -p \
      WEBHOOK_URL=${hookUrl} \
      FILE_NAME=${fileName} \
      IMAGE=${imageURL}:latest \
      BUILD_NUM=${currentBuild.number} \
      -n ${project} | oc create -f - -n ${project}
    """

    // Block and wait for job to return
    // Job will return the name of the pod
    print "Waiting for ${fileName}.jmx tests to complete..."
    jobPod = waitForWebhook hook
    print "POST to webhook recieved from test suite pod: ${jobPod}"
   
    try {
      // Get reports directory from pod for HTML report dashboard
      sh """
        oc rsync ${jobPod}:/jmeter/${fileName}-reports ${workspace} -n ${project}
      """

      // Use HTML Publisher plugin to display reports
      publishHTML (target: [
          allowMissing: true,
          alwaysLinkToLastBuild: false,
          keepAll: true,
          reportDir: "${workspace}/${fileName}-reports",
          reportFiles: "index.html",
          reportName: "${fileName} Report"
        ])
    } catch(err){
      print "An exception occurred while parsing the JMeter reports: ${err}"
    } finally {
      sh """
        oc delete job jmeter-test-suite -n ${project} --ignore-not-found=true
        oc delete pods -l jobName=jmeter-test-suite -n ${project} --ignore-not-found=true
      """
    } // finally
  } // for loop
} // node