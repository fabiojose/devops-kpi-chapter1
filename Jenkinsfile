pipeline {
  agent {
    label 'docker'
  }

  stages {
    stage('Setup') {
      steps {
        script {
          if (env.GIT_BRANCH == 'origin/develop' || env.GIT_BRANCH ==~ /(.+)feature-(.+)/) {
            target = 'dev'
          } else if (env.GIT_BRANCH ==~ /(.+)release-(.+)/) {
            target = 'pre'
          } else if (env.GIT_BRANCH == 'origin/master') {
            target = 'pro'
          } else {
            error "Unknown branch type: ${env.GIT_BRANCH}"
          }

          appversion  = sh(script: 'echo $(cat src/package.json | grep version | cut -d\' \' -f4 | sed \'s|[",]||g\')', returnStdout: true).trim()
          version     = appversion.take(10) + '-' + env.BUILD_NUMBER
          appname     = env.JOB_NAME
          prjname     = 'oss-opensource'
          packname    = appname + '-v' + version + '.tar.gz'
          publish_url = env.NEXUS_URL + '/repository/raw-nodejs/' + prjname + '/' + packname
        }
      }
    }

    stage('APP Build') {
      steps {
        sh 'node --version'
        sh 'npm --version'
        sh 'cd src && npm install'
      }
    }

    stage('APP Quality'){
      steps {
        withEnv(["PATH+SONAR=${tool 'sonarqube-scanner-v3.0.3'}/bin"]) {
          withCredentials([usernamePassword(credentialsId: 'sonarqube-credential', passwordVariable: 'SONARQUBE_PASSWORD', usernameVariable: 'SONARQUBE_USERNAME')]) {
            sh """sonar-scanner \
                      -Dsonar.sources=src \
                      -Dsonar.projectKey=${env.JOB_NAME} \
                      -Dsonar.projectName=${env.JOB_NAME} \
                      -Dsonar.projectBaseDir=${env.WORKSPACE} \
                      -Dsonar.login=$SONARQUBE_PASSWORD \
                      -Dsonar.host.url=${env.SONARQUBE_URL} \
                      -Dsonar.exclusions=src/test/**,src/node_modules/**,**/migrations/**,**/libs/transactional_db/** \
                      -Dsonar.sourceEncoding=UTF-8 \
                      -Dsonar.language=js \
                      -Dsonar.projectDescription='OSS OpenSource - API Gateway' \
                      -Dsonar.projectVersion=${version}
               """
          }
        }
      }
    }

    stage('APP Unit Test') {
      steps {
        sh 'cd src && npm test'
        junit 'src/mocha-report.xml'
      }
    }

    stage('APP Package'){
      steps {
        sh "tar --exclude='./.git' --exclude='./Jenkinsfile' --exclude='./Dockerfile' -czv src/ -f " + packname
      }
    }

    stage('APP Publish') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-credential', passwordVariable: 'NEXUS_PASSWORD', usernameVariable: 'NEXUS_USERNAME')]) {
          sh 'curl -v -u $NEXUS_USERNAME:$NEXUS_PASSWORD --upload-file ' + packname + ' ' + publish_url
        }
      }
    }

    stage('Docker Build') {
      steps {
        script {
          docker.build(appname, "-f src/Dockerfile src")
        }
      }
    }

    stage('Docker Publish') {
      steps {
        script {
          docker.withRegistry(env.REGISTRY_URL) {
            docker.image(appname).push(version)
            docker.image(appname).push("latest")
          }
        }
      }
    }

    stage('DEV Deploy') {
      steps {
        sh 'echo Deploy to DEV . . .'
      }
    }

    stage('PRE Deploy') {
      when {
        expression { return target == 'pre' || target == 'pro' }
      }
      steps {
        sh 'echo Deploy to PRE . . .'
      }
    }

    stage('UAT Test') {
      when {
        expression { target == 'pro' }
      }
      steps {
        sh 'echo UAT Test . . .'
      }
    }

    stage('Approval') {
      when {
        expression { target == 'pro' }
      }
      steps {
        timeout(time:30, unit:'MINUTES') {
          input message: "Deploy to Production?", id: "approval"
        }
      }
    }

    stage('PRO Deploy') {
      when {
        expression { return target == 'pro' }
      }
      steps {
        sh 'echo Deploy to PRO . . .'
      }
    }
  }

  post {
    success {
      script {
        currentBuild.result = "SUCCESS"

        /* Custom data map for InfluxDB */
        def custom = [:]
        custom['branch']      = env.GIT_BRANCH
        custom['environment'] = target
        custom['part']        = 'jenkins'
        custom['version']     = version

        step([$class: 'InfluxDbPublisher', customData: custom, target: 'devops-kpi'])
      }
    }

    failure {
      script {
        currentBuild.result = "FAILURE"

        /* Custom data map for InfluxDB */
        def custom = [:]
        custom['branch']      = env.GIT_BRANCH
        custom['environment'] = target
        custom['part']        = 'jenkins'
        custom['version']     = version

        step([$class: 'InfluxDbPublisher', customData: custom, target: 'devops-kpi'])
      }
    }

    unstable {
      script {
        currentBuild.result = "FAILURE"

        /* Custom data map for InfluxDB */
        def custom = [:]
        custom['branch']      = env.GIT_BRANCH
        custom['environment'] = target
        custom['part']        = 'jenkins'
        custom['version']     = version

        step([$class: 'InfluxDbPublisher', customData: custom, target: 'devops-kpi'])
      }
    }

    aborted {
      script {
        currentBuild.result = "FAILURE"

        /* Custom data map for InfluxDB */
        def custom = [:]
        custom['branch']      = env.GIT_BRANCH
        custom['environment'] = target
        custom['part']        = 'jenkins'
        custom['version']     = version

        step([$class: 'InfluxDbPublisher', customData: custom, target: 'devops-kpi'])
      }
    }
  }
}
