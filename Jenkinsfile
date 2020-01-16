node {
  withEnv([
    'APP_HOSTNAME= fusetest.apps.met-police.poc-optimus.co.uk',
    'APP_NAME= fusetest',
    'USER_NAME= kubeadmin',
    'OSE_SERVER= https://api.met-police.poc-optimus.co.uk:6443',
    'DEVEL_PROJ_NAME= test-fuse-application',
    'DEPLOYCONFIG= s2i-fuse75-spring-boot-camel-xml']){
    
    stage('Fuse Deploy') { 
     // sh """
     // oc login -u${USER_NAME} -p${USER_PASSWD} --server=${OSE_SERVER}
     // oc project ${DEVEL_PROJ_NAME}
     // """
      def BUILD_CONFIG = sh (script: "oc get bc | grep \$DEPLOYCONFIG | awk '{print \$1}'",returnStdout: true).trim()
      sh "echo ${BUILD_CONFIG}"
      if (!BUILD_CONFIG?.trim()) {
        println "Create a new app";
        // comment these lines to test the pipeline
        sh """
          oc new-app --template="openshift/s2i-fuse75-spring-boot-camel-xml"
          echo "Find build id"
        """
        // comment above lines to tes the script
        def BUILD_ID = sh (script: "oc get builds | tail -1 | awk '{print \$1}'",returnStdout: true).trim()
        def rc=1
        def attempts=5
        def count=0
        while((rc!=0)&&(count<attempts)){
          BUILD_ID = sh (script: "oc get builds | tail -1 | awk '{print \$1}'",returnStdout: true).trim()
          if(BUILD_ID=="NAME"){
            count=count+1;
            println "Attempt ${count}/${attempts}"
            sleep(5)
          }
          else{
            rc=0;
            println "Build ID is: ${BUILD_ID}";
          }
        }
        if (rc!=0){
          println "Fail: Build could not be found after maximum attempts"
          return;
        }
      }
      else{
        println "App Exists. Triggering application build and deployment"
        println "BUILD_config is: ${BUILD_CONFIG}"
        //comment this line to test pipeline
        sh """oc start-build ${BUILD_CONFIG}"""
      }
      println "Waiting for build to start"
      def rc=1;
      def attempts=5;
      def count=0;
      def BUILD_ID = sh (script: "oc get builds | tail -1 | awk '{print \$1}'",returnStdout: true).trim()
      while((rc!=0)&&(count<attempts)){
        def status = sh (script: "oc get build ${BUILD_ID} --template='{{.status.phase}}'",returnStdout: true).trim()
        if ((status == "Failed") || (status == "Error") || (status == "Canceled")){
          println "Fail: Build completed with unsuccessful status: ${status}"
          return;
        }
        if(status == "Complete"){
          println "Build completed successfully, will test deployment next"
          rc=0;
        }
        if(status == "Running"){
          echo "Build started"
          rc=0;
        }
        if(status == "Pending"){
          count=count+1;
          println "Attempt ${count}/${attempts}"
          sleep(5)
        }
      } //end of waiting for build to start
      //comment this line to test pipeline
      sh """oc build-logs ${BUILD_ID}"""
      println "Checking build result status"
      rc=1;
      count=0;
      attempts=5;
      while((rc!=0)&&(count<attempts)){
        def status = sh (script: "oc get build ${BUILD_ID} --template='{{.status.phase}}'",returnStdout: true).trim()
        if ((status == "Failed") || (status == "Error") || (status == "Canceled" )){
          println "Fail: Build completed with unsuccessful status: ${status}"
          return;
        }
        if (status == "Complete"){
          println "Build completed successfully, will test deployment next"
          rc=0;
        }
        else{
          count=count+1
          println "Attempt ${count}/${attempts}"
          sleep(5)
        }
      }
      if(rc!=0){
        println "Fail: Build did not complete in a reasonable period of time"
        return
      }
    } //end of stage fuse deploy
    stage('Fuse Test Deployment') {
      //Scale up the test deployment
      def RC_ID = sh (script: "oc get rc | tail -1 | awk '{print \$1}'",returnStdout: true).trim()
      println "RC_ID: ${RC_ID}"
      def podname = sh (script: "oc get po | grep Running |awk '{print \$1}'",returnStdout: true).trim()
      def ipaddress = sh (script: "oc describe pod ${podname} | grep IP:|awk '{print \$2}'",returnStdout: true).trim()
      println "ip address is : ${ipaddress}"
      def statuscode = sh (script: "curl -Is --connect-timeout 2 ${ipaddress}:8081/health|head -1|awk '{print \$2}'",returnStdout: true).trim()
      println "Scaling up new deployment"
      //comment the below line to test the pipeline
      sh"""oc scale --replicas=1 rc ${RC_ID}"""
      println "Checking for successful test deployment at ${HOSTNAME}"
      def rc=1
      def count=0
      def attempts=5
      while((rc!=0)&&(count<attempts)){
        if(statuscode == "200"){
          rc=0;
          println "test deployment successfull with statuscode: ${statuscode}"
          break;
        }
        count=count+1;
        echo "Attempt ${count}/${attempts}"
        sleep(5)
      }
      if(rc!=0){
        echo "Failed to access test deployment, aborting roll out."
        return
      }
    }
  }  
}
