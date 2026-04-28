
/*
  Jenkinsfile (Enterprise Version)
  --------------------------------
  This Jenkinsfile is an evolution of Jenkinsfile.simple.

  Why this version exists:
  - Corporate laptops restrict kubectl / Docker locally
  - Validation and deployment must run in CI (AKS-hosted Jenkins)
  - Real enterprise pipelines require:
      * Multi-repo checkout (infra + app code)
      * Gated environment promotions (QA / PROD)
      * Smoke tests and UAT hooks
      * Reusable scripts instead of inline shell
      * CI-based Kubernetes validation

  Relationship to Jenkinsfile.simple:
  - Jenkinsfile.simple = MVP / learning baseline
  - Jenkinsfile        = Production-ready evolution
*/

pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  parameters {
    choice(name: 'TARGET_ENV', choices: ['dev','qa','uat','prod'], description: 'Target environment')
    string(name: 'FRONTEND_REPO', defaultValue: 'https://github.com/<your-username>/clock-frontend.git', description: 'Frontend repo URL')
    string(name: 'BACKEND_REPO',  defaultValue: 'https://github.com/<your-username>/clock-backend.git',  description: 'Backend repo URL')
    string(name: 'FRONTEND_BRANCH', defaultValue: 'main', description: 'Frontend branch')
    string(name: 'BACKEND_BRANCH',  defaultValue: 'main', description: 'Backend branch')
    string(name: 'REGISTRY', defaultValue: 'myacr.azurecr.io', description: 'Container registry host (ACR later)')
  }

  environment {
    IMAGE_TAG = "${BUILD_NUMBER}"
    // Sonar URL will be reachable inside AKS later (service name / ingress)
    SONAR_HOST_URL = "http://sonarqube:9000"
    // Kustomize lives in kubectl since v1.14; we’ll use kubectl apply -k
    K8S_DIR = "k8s/overlays"
  }

  stages {

    stage('0) Preflight (CI-only validation)') {
      steps {
        echo "Local kubectl is restricted; validation will run on Jenkins agent in AKS."
        sh '''
          echo "Repo: clock-infra"
          ls -la
          echo "Checking kustomize directories exist..."
          test -d ${K8S_DIR}/dev
          test -d ${K8S_DIR}/qa
          test -d ${K8S_DIR}/uat
          test -d ${K8S_DIR}/prod
        '''
      }
    }

    stage('1) Checkout') {
      steps {
        // Checkout infra (this repo)
        checkout scm

        // Checkout app repos into workspace subfolders
        dir('clock-frontend') {
          checkout([$class: 'GitSCM',
            branches: [[name: "*/${params.FRONTEND_BRANCH}"]],
            userRemoteConfigs: [[url: params.FRONTEND_REPO, credentialsId: 'GIT_CREDS']]
          ])
        }
        dir('clock-backend') {
          checkout([$class: 'GitSCM',
            branches: [[name: "*/${params.BACKEND_BRANCH}"]],
            userRemoteConfigs: [[url: params.BACKEND_REPO, credentialsId: 'GIT_CREDS']]
          ])
        }
      }
    }

    stage('2) Sonar quality check') {
      steps {
        // Typical Jenkins+Sonar pattern: token stored in Jenkins Credentials and used by scanner. [2](https://www.youtube.com/watch?v=G9MmLUsBd3g)[3](https://ustglobal-my.sharepoint.com/personal/u38764_ust_com/Documents/Microsoft%20Teams%20Chat%20Files/StageCraft-JF-node-dummy-refwf-node-dummy-verbatim-template.pdf?web=1)
        withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
          sh '''
            echo "Sonar scan - Frontend (React)"
            cd clock-frontend
            # Use npx sonar-scanner pattern (common in Node pipelines) [3](https://ustglobal-my.sharepoint.com/personal/u38764_ust_com/Documents/Microsoft%20Teams%20Chat%20Files/StageCraft-JF-node-dummy-refwf-node-dummy-verbatim-template.pdf?web=1)
            npx --yes sonar-scanner \
              -Dsonar.host.url=${SONAR_HOST_URL} \
              -Dsonar.token=${SONAR_TOKEN} \
              -Dsonar.projectKey=clock-frontend \
              -Dsonar.sources=src

            echo "Sonar scan - Backend (Python)"
            cd ../clock-backend
            # Placeholder: you can add sonar-project.properties for Python later
            # Keeping a simple scan hook for future expansion
          '''
        }
      }
    }

    stage('3) Docker build -> image') {
      steps {
        // In your enterprise pipelines, Docker/build steps are commonly scripted with tags including BUILD_NUMBER. [4](https://ustglobal-my.sharepoint.com/personal/u38764_ust_com/Documents/Microsoft%20Teams%20Chat%20Files/BA-OP-CDMDPD-JF-3diffRepos-II.pdf?web=1)[5](https://ustglobal.sharepoint.com/teams/P4-GitHubGitLift/Shared%20Documents/StageCraft%20Testing/Stagecraft-jenkins-to-git.pdf?web=1)
        sh '''
          echo "Building images with tag: ${IMAGE_TAG}"
          docker build -t ${REGISTRY}/clock-frontend:${IMAGE_TAG} ./clock-frontend
          docker build -t ${REGISTRY}/clock-backend:${IMAGE_TAG}  ./clock-backend
        '''
      }
    }

    stage('4) Upload docker image to registry') {
      steps {
        // Registry creds managed via Jenkins credential store (standard practice). [1](https://ustglobal.sharepoint.com/teams/P4-GitHubGitLift/Shared%20Documents/StageCraft%20Testing/Stagecraft-jenkins-to-git.pdf?web=1)[6](https://ustglobal-my.sharepoint.com/personal/u38764_ust_com/Documents/Microsoft%20Teams%20Chat%20Files/StageCraft-JF-node-dummy-refwf-node-dummy-verbatim-template.pdf?web=1)
        withCredentials([usernamePassword(credentialsId: 'REGISTRY_CREDS', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
          sh '''
            echo "$REG_PASS" | docker login ${REGISTRY} -u "$REG_USER" --password-stdin
            docker push ${REGISTRY}/clock-frontend:${IMAGE_TAG}
            docker push ${REGISTRY}/clock-backend:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('5) Dev deploy (config)') {
      when { expression { params.TARGET_ENV == 'dev' } }
      steps {
        sh '''
          ./scripts/set-images.sh dev ${REGISTRY} ${IMAGE_TAG}
          ./scripts/deploy-kustomize.sh dev
        '''
      }
    }

    stage('6) SMOKE test') {
      when { expression { params.TARGET_ENV == 'dev' } }
      steps {
        sh '''
          ./scripts/smoke-test.sh dev
        '''
      }
    }

    stage('7) QA promotion | gated approval') {
      when { expression { params.TARGET_ENV == 'qa' } }
      steps {
        input message: "Promote build ${IMAGE_TAG} to QA?"
      }
    }

    stage('8) QA deploy (config)') {
      when { expression { params.TARGET_ENV == 'qa' } }
      steps {
        sh '''
          ./scripts/set-images.sh qa ${REGISTRY} ${IMAGE_TAG}
          ./scripts/deploy-kustomize.sh qa
        '''
      }
    }

    stage('9) UAT test automated test suites') {
      when { expression { params.TARGET_ENV == 'uat' } }
      steps {
        sh '''
          ./scripts/set-images.sh uat ${REGISTRY} ${IMAGE_TAG}
          ./scripts/deploy-kustomize.sh uat

          echo "Running UAT automation hook..."
          ./scripts/uat-tests.sh uat
        '''
      }
    }

    stage('10) Prod approval') {
      when { expression { params.TARGET_ENV == 'prod' } }
      steps {
        input message: "Deploy build ${IMAGE_TAG} to PROD?"
      }
    }

    stage('11) Prod deploy config') {
      when { expression { params.TARGET_ENV == 'prod' } }
      steps {
        sh '''
          ./scripts/set-images.sh prod ${REGISTRY} ${IMAGE_TAG}
          ./scripts/deploy-kustomize.sh prod
        '''
      }
    }
  }
}

