pipeline {
  agent {
    kubernetes {
      label 'ci-sample'
      yamlFile 'builder-pod.yaml'
    }
  }
  environment {
    ROLE_ID = "94cf2daf-77a5-d475-8475-713ee90e31e3"
  }
  stages {
    stage('Build') {
      steps {
        container('gradle') {
          sh """
            gradle clean bootJar
          """
        }
      }
      post {
        success {
          sh """
            ls -la build/libs
          """
          stash includes: 'build/libs/ci-sample*.jar', name: 'app'
        }
      }
    }
    stage('Containerize') {
      steps {
        container('docker') {
          unstash 'app'
          script {
            def image = docker.build("nthienan/ci-sample:${env.BUILD_NUMBER}")
            docker.withRegistry( '', 'nthienan_dockerhub') {
              image.push "${env.BUILD_NUMBER}"
              image.push "latest"
            }
          }
        }
      }
    }
    stage('Deploy') {
      steps {
        container('vault') {
          withCredentials([string(credentialsId: 'jenkins_vault_token', variable: 'VAULT_TOKEN')]) {
            sh """
              set +x
              echo "Generating configurations ..."
              export VAULT_ADDR=http://vault.nthienan.com
              SECRET_ID=`vault write -field=secret_id -f auth/approle/role/team-a/secret-id`
              VAULT_TOKEN=`vault write -field=token auth/approle/login role_id=$ROLE_ID secret_id=\$SECRET_ID`
              USERNAME=`vault kv get -field=username kv/team-a/mongodb`
              PASSWORD=`vault kv get -field=password kv/team-a/mongodb`

              config_file="./application.properties"
              cp src/main/resources/application.properties.template \$config_file
              sed -i -e 's,MONGO_HOST,'13.251.129.107',g' \$config_file
              sed -i -e 's,MONGO_USER,'\$USERNAME',g' \$config_file
              sed -i -e 's,MONGO_PASSWORD,'\$PASSWORD',g' \$config_file
            """
          }
          stash includes: 'application.properties', name: 'config'
        }
        container('alpine') {
          unstash 'config'
          withCredentials([usernamePassword(credentialsId: 'ssh_depoyment_server', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
            script {
              def aws_server = [:]
              aws_server.name = "aws_server"
              aws_server.host = "13.251.129.107"
              aws_server.allowAnyHosts = true
              aws_server.user = USERNAME
              aws_server.password = PASSWORD
              sshRemove remote: aws_server, path: "./application.properties", failOnError: false
              sshPut remote: aws_server, from: './application.properties', into: '/home/ubuntu/application.properties'
              sshCommand remote: aws_server, command: "docker rm -f ci-sample || true && docker run -d --name ci-sample -p 80:8080 -v /home/ubuntu/application.properties:/tmp/application.properties -e OPTS='--spring.config.location=file:/tmp/application.properties' nthienan/ci-sample:${env.BUILD_NUMBER}"
            }
          }
        }
      }
      post {
        success {
          archiveArtifacts artifacts: 'application.properties'
        }
      }
    }
  }
}
