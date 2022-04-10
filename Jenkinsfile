pipeline {
  options {
    disableConcurrentBuilds()
    skipDefaultCheckout(true)
  }
//  triggers {  
//    upstream(upstreamProjects: "docs", threshold: hudson.model.Result.SUCCESS)
//  }
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: ubuntu
            image: robinhoodis/ubuntu:latest
            imagePullPolicy: Always
            command:
            - cat
            tty: true
          - name: kaniko
            image: gcr.io/kaniko-project/executor:debug
            imagePullPolicy: IfNotPresent
            command:
            - /busybox/cat
            tty: true
            volumeMounts:
              - name: kaniko-secret
                mountPath: /kaniko/.docker
          restartPolicy: Never
          volumes:
            - name: kaniko-secret
              secret:
                secretName: regcred
                items:
                  - key: .dockerconfigjson
                    path: config.json
        '''
    }
  }
  stages {
    stage('INIT') {
      steps {
        cleanWs()
        checkout scm
        echo "Building ${env.JOB_NAME}..."
      }
    }
    stage('Dockerfile change') {
      when {
        changeset "Dockerfile"
      }
      steps {
        container('ubuntu') {
          sh 'sh increment-version.sh'
        }
        sh 'git config user.email "robin@mordasiewicz.com"'
        sh 'git config user.name "Robin Mordasiewicz"'
        // sh 'git add -u'
        // sh 'git diff --quiet && git diff --staged --quiet || git commit -m "`cat VERSION`"'
        sh 'git add . && git diff --staged --quiet || git commit -m "`cat VERSION`"'
        withCredentials([gitUsernamePassword(credentialsId: 'github-pat', gitToolName: 'git')]) {
          // sh 'git diff --quiet && git diff --staged --quiet || git push origin HEAD:main'
          // sh 'git diff --quiet HEAD || git push origin HEAD:main'
          sh 'git push origin HEAD:main'
        }
      }
    }
    stage('Check repo for container') {
      when { changeset "VERSION"}
      steps {
        container('ubuntu') {
          sh 'skopeo inspect docker://docker.io/robinhoodis/nginx:`cat VERSION` > /dev/null || echo "create new container: `cat VERSION`" > BUILDNEWCONTAINER.txt'
        }
      }
    }
    stage('Build/Push Container') {
      when { changeset "VERSION"}
      steps {
        container(name: 'kaniko', shell: '/busybox/sh') {
          script {
            sh ''' 
            [ ! -f BUILDNEWCONTAINER.txt ] || \
            /kaniko/executor --dockerfile=Dockerfile \
                             --context=git://github.com/robinmordasiewicz/nginx.git \
                             --destination=robinhoodis/nginx:`cat VERSION` \
                             --destination=robinhoodis/nginx:latest \
                             --cache=true
            '''
          }
        }
      }
    }
    stage('create new manifest') {
      when { changeset "VERSION"}
      steps {
        sh 'mkdir argocd'
        dir ( 'argocd' ) {
          git branch: 'main', url: 'https://github.com/robinmordasiewicz/argocd.git'
          sh 'sh increment-nginx-version.sh'
        }
      }
    }
    stage('commit new manifest') {
      when { changeset "VERSION"}
      steps {
        dir ( 'argocd' ) {
          sh 'git config user.email "robin@mordasiewicz.com"'
          sh 'git config user.name "Robin Mordasiewicz"'
          sh 'git add .'
          sh 'git diff --quiet && git diff --staged --quiet || git commit -am "NGINX Manifest `cat nginx/VERSION`"'
          withCredentials([gitUsernamePassword(credentialsId: 'github-pat', gitToolName: 'git')]) {
            sh 'git diff --quiet && git diff --staged --quiet || git push origin main'
          }
        }
      }
    }
  }
  post {
    always {
      cleanWs(cleanWhenNotBuilt: false,
            deleteDirs: true,
            disableDeferredWipeout: true,
            notFailBuild: true,
            patterns: [[pattern: '.gitignore', type: 'INCLUDE'],
                       [pattern: '.propsfile', type: 'EXCLUDE']])
    }
  }
}
