properties([buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '5')), parameters([string(defaultValue: 'AP5P5hu1iVpC7PnPLwoSg6YCs3k', description: '', name: 'SERVER_ID')]), pipelineTriggers([pollSCM('* * * * *')])])
node ('master') {
   
   def mvnHome
   ansiColor('xterm') {
      
      timestamps {

         withSonarQubeEnv {
         
            stage('Preparation') { scmcheckout() }
            stage ('Maven Build') { build() }
            stage ('Unit Test') { unitTest() }
            stage ('SonarQube Analysis') { sonarQubeAnalysis() }
            //stage ('Publish Results') { Results() }
            stage ('Upload Artifact') { UploadArtifact() }
            stage ('Deploy 2 QA') { deployQA() }
            step([$class: 'WsCleanup'])
            stage ('Deploy 2 Production') { deployproduction() }

         }     
      
      }


   }
   

}

def scmcheckout() {
   checkout scm
   //git 'https://github.com/mybatis/jpetstore-6.git'
   mvnHome = tool 'M2'
   
}
def sonarQubeAnalysis() {
   
   sh "'${mvnHome}/bin/mvn' org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar"
}
def build() {
   
   // Run the maven build
   //sh "sed -i -P 's/0.1-SNAPSHOT/0.1-SNAPSHOT.${BUILD_NUMBER}/g' pom.xml"
   if (isUnix()) {
      sh "'${mvnHome}/bin/mvn' -Dmaven.test.failure.ignore clean package"
   } else {
      bat(/"${mvnHome}\bin\mvn" -Dmaven.test.failure.ignore clean package/)
   }

}
def runTests() {
  try { 

    setTestStatus(sh (returnStatus: true, script: "'${mvnHome}/bin/mvn' test"))

  }
  finally {

    junit '**/target/surefire-reports/TEST-*.xml'
    archive 'target/*.war'

  }


}
@NonCPS
def setTestStatus(testStatus) {
  if (testStatus == 0) {
    currentBuild.result = 'SUCCESS'
  } else {
    def testResult = currentBuild.rawBuild.getAction(hudson.tasks.junit.TestResultAction.class)
    currentBuild.result = (testResult != null && testResult.failCount > 0) ? 'UNSTABLE' : 'FAILURE'
  }
}
def Results() {
   
   junit '**/target/surefire-reports/TEST-*.xml'
   archive 'target/*.war'

}
def unitTest() {
  try {
        // Any maven phase that that triggers the test phase can be used here.
        sh "'${mvnHome}/bin/mvn' test"
    } catch(err) {
        step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
        throw err
    }
}
def UploadArtifact() {

    echo 'printing Build Stamp on Artifact'
    sh 'mv $WORKSPACE/target/jpetstore.war $WORKSPACE/target/jpetstore.0.1.${BUILD_NUMBER}-SNAPSHOT.war'
    fingerprint 'target/**.war'
    sh 'ls -lart target/'
    def server = Artifactory.server "$SERVER_ID"
    def buildInfo = Artifactory.newBuildInfo()
    buildInfo.env.capture = true
    buildInfo.env.collect()

    def uploadSpec = """{
      "files": [
        {
          "pattern": "target/**.jar",
          "target": "libs-snapshot-local/${JOB_NAME}/${BUILD_NUMBER}/"
        }, {
          "pattern": "target/*.pom",
          "target": "libs-snapshot-local/${JOB_NAME}/${BUILD_NUMBER}/"
        }, {
          "pattern": "target/*.war",
          "target": "libs-snapshot-local/${JOB_NAME}/${BUILD_NUMBER}/"
        }
      ]
    }"""
    // Upload to Artifactory.
    server.upload spec: uploadSpec, buildInfo: buildInfo

    buildInfo.retention maxBuilds: 10, maxDays: 7, deleteBuildArtifacts: true
    // Publish build info.
    server.publishBuildInfo buildInfo
}
def deployQA() {
    parallel(longerTests: {
        runTests(30)
    }, quickerTests: {
        runTests(20)
    })
   
   mail bcc: '', body: "Successfully Deployed to QA. \n Please go to below URL and provide your input to Proceed or Abort to deploy. \n ${BUILD_URL}input", cc: 'babupeddada@gmail.com', from: '', replyTo: '', subject: "job ${JOB_NAME} build number ${BUILD_NUMBER} status ", to: 'babupeddada@gmail.com, peddadabrp@gmail.com'
    
}
def runTests(duration) {
    node {
        sh "sleep ${duration}"
        }
}
def deployproduction() {
  
  node ('tomcat') {

   input message: "Does production look good?"
   step([$class: 'WsCleanup'])
   def server = Artifactory.server "${SERVER_ID}"
    def downloadSpec = """{
        "files": [
         {
            "pattern": "libs-snapshot-local/${JOB_NAME}/${BUILD_NUMBER}/jpetstore.0.1.${BUILD_NUMBER}-SNAPSHOT.war",
            "target": "$WORKSPACE/"
           }
    ]
  }"""
  server.download(downloadSpec)
  sh 'cd /opt/apache-tomcat-7.0.73/bin/ && echo jenkins | sudo -S ./shutdown.sh'
  sh 'cd /opt/apache-tomcat-7.0.73/webapps/ && echo jenkins | sudo -S rm -rf jpetstore.war jpetstore && mv $WORKSPACE/${JOB_NAME}/${BUILD_NUMBER}/jpetstore.0.1.${BUILD_NUMBER}-SNAPSHOT.war $WORKSPACE/jpetstore.war'
  sh 'echo jenkins | sudo -S cp -R $WORKSPACE/jpetstore.war /opt/apache-tomcat-7.0.73/webapps/'
  sh 'echo jenkins | sudo -S chown root:root /opt/apache-tomcat-7.0.73/webapps/jpetstore.war'
  sh 'cd /opt/apache-tomcat-7.0.73/bin/ && echo jenkins | sudo -S ./startup.sh'
  step([$class: 'WsCleanup'])

  }

}
