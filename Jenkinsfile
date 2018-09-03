def jobnameparts = JOB_NAME.tokenize('/') as String[]
def jobconsolename = jobnameparts[1]

script {
   System.setProperty("org.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL", "300");
}

properties (
    [
        [$class: 'BuildDiscarderProperty',strategy: [$class: 'LogRotator', numToKeepStr: '10']]
    ]
)


stage('Docker build') {

  parallel (
    "base": {
      def label = "mypod-${jobconsolename}-${UUID.randomUUID().toString()}".take(63)

      podTemplate(
        label: label,
        nodeUsageMode: 'EXCLUSIVE',
        podRetention: never(),
        serviceAccount: 'jenkins',
        annotations: [ podAnnotation(key: "iam.amazonaws.com/role", value: "infra-kube1-jenkins-pod-slave")],
        containers: [containerTemplate(image: '556593845588.dkr.ecr.eu-west-1.amazonaws.com/jenkins-worker-base:latest',name: 'docker', command: 'cat', ttyEnabled: true)],
        volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
        ) {
                  node(label) {
                      withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "jenkins_${jobconsolename}_aws"]]) {
                        withEnv(['AWS_REGION=eu-west-1']) {
                            container('docker') {
                              checkout scm
                              sh "apk -Uuv add python3 go gcc musl-dev openssl"
                              sh "make"
                              sh "eval \$(aws ecr get-login --no-include-email --region eu-west-1)"
                                try {
                                  sh "aws ecr --region eu-west-1 create-repository --repository-name logstash-oss"
                                } catch (e) {
                                  echo "Registry already exists"
                              }
                              sh "docker tag docker.elastic.co/logstash/logstash-oss:${BRANCH_NAME} 556593845588.dkr.ecr.eu-west-1.amazonaws.com/logstash-oss:${BRANCH_NAME}"
                              sh "docker push 556593845588.dkr.ecr.eu-west-1.amazonaws.com/logstash-oss:${BRANCH_NAME}"
                            } //container
                        } //env
                      } //cred
                  } //node
      } //pod
    } // base
  ) // parallel
} //stage build
