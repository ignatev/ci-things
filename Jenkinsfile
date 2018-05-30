pipeline {

  agent { label 'master' }

  triggers { pollSCM('0-59 * * * *') }

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '2'))
  }

  stages {
    stage('Build sbt dependecies') {
      //when { branch 'master' }
      agent {
        docker {
          image 'hseeberger/scala-sbt:8u141-jdk_2.12.3_0.13.16'
          args '-u root -v /var/cache/.ivy2:/root/.ivy2'
        }
      }
      steps {
        withCredentials([file(credentialsId: 'nexus_ca_cert', variable: 'NEXUS_CA_CERT'),
                         string(credentialsId: 'nexus_ca_cert_password', variable: 'NEXUS_CA_CERT_PASSWORD'),
                         file(credentialsId: 'sbt_nexus_credentials', variable: 'SBT_CREDENTIALS')]) {
            sh 'cp ${NEXUS_CA_CERT} $JAVA_HOME/jre/lib/security'
            sh ' keytool -v -importkeystore ' +
               '-srckeystore \${JAVA_HOME}/jre/lib/security/artifactory.p12 ' +
               '-srcstoretype PKCS12 ' +
               '-srcstorepass ${NEXUS_CA_CERT_PASSWORD} ' +
               '-destkeystore \${JAVA_HOME}/jre/lib/security/cacerts ' +
               '-deststoretype JKS ' +
               '-deststorepass changeit'

            sh 'cp ${SBT_CREDENTIALS} \$HOME/.ivy2/.my-credentials'
            dir('dir_name') {
              sh "SBT_OPTS='-Xms512m -Xmx2048m' sbt package"
            }
        }
      }
    }

    stage('build docker images') {
     // when { branch 'master' }
      agent {
        docker {
          image 'ignatev/dind-jenkins:0.2'
          args '-u root -v /var/run/docker.sock:/var/run/docker.sock -v /var/cache/gradle:/tmp/gradle-user-home:rw'
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'ci_nexus_u_p', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD'),
                         file(credentialsId: 'nexus_ca_cert', variable: 'NEXUS_CA_CERT'),
                         string(credentialsId: 'nexus_ca_cert_password', variable: 'NEXUS_CA_CERT_PASSWORD')
        ]) {
            sh 'cp ${NEXUS_CA_CERT} $JAVA_HOME/jre/lib/security'
            sh ' keytool -v -importkeystore ' +
              '-srckeystore \${JAVA_HOME}/jre/lib/security/artifactory.p12 ' +
              '-srcstoretype PKCS12 ' +
              '-srcstorepass ${NEXUS_CA_CERT_PASSWORD} ' +
              '-destkeystore \${JAVA_HOME}/jre/lib/security/cacerts ' +
              '-deststoretype JKS ' +
              '-deststorepass changeit'
            sh 'echo nexusUsername=${NEXUS_USERNAME} >> gradle.properties'
            sh 'echo nexusPassword=${NEXUS_PASSWORD} >> gradle.properties'
            sh "GRADLE_USER_HOME=/tmp/gradle-user-home gradle -Pci-build :azure-service:docker :job-processing:docker :job-rest-api:docker -x test --parallel"
        }
        script {
          PROJECT_VERSION = sh (
            script: "gradle properties -q | grep \"version:\" | awk '{print \$2}'",
            returnStdout: true
          ).trim()
          sh "echo Building project in version: $PROJECT_VERSION"

        }
      }
    }

    stage('Publish to aksregistryprod.azurecr.io') {
      steps {
        sh "echo $PROJECT_VERSION"
        script {
          docker.withRegistry('https://registryname', 'sercret_name') {
            docker.image("registryname/job-rest-api:$PROJECT_VERSION").push()
            docker.image("registryname/job-processing:$PROJECT_VERSION").push()
            docker.image("registryname/azure-dispatcher-service:$PROJECT_VERSION").push()
          }
        }
      }
    }

    stage('Deploy') {
      agent {
        docker {
          image 'ignatev/kubectl:1.9.8'
          args '-u root'
        }
      }
      steps {
        withCredentials([ string(credentialsId: 'temp-namespace-jenkins-service-account', variable: 'TOKEN') ]) {
          sh 'kubectl config set-context dev --cluster=dev --user=jenkins --namespace=sak-spark-aas'
          sh 'kubectl config use-context dev'
          sh 'kubectl config set-credentials jenkins --token=$TOKEN'
          sh 'kubectl config set-cluster dev --server=https://cluster-name.hcp.eastus.azmk8s.io --insecure-skip-tls-verify=true'
          sh 'kubectl auth can-i create deployments --namespace staging'
          sh 'kubectl auth can-i create deployments --namespace dev'
        }
      }
    }
  }
}
