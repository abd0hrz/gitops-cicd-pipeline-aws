pipeline {
    agent {
        kubernetes {
            label 'nodejs-pipeline'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-kaniko-sa
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - sleep
    args:
    - 99d
    env:
    - name: AWS_DEFAULT_REGION
      value: "us-east-1"
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/workspace
  - name: git
    image: alpine/git:latest
    command:
    - sleep
    args:
    - 99d
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/workspace
  - name: jnlp
    image: jenkins/inbound-agent:3309.v27b_9314fd1a_4-1
    env:
    - name: JENKINS_URL
      value: http://jenkins.jenkins.svc.cluster.local:8080/
    - name: JENKINS_TUNNEL
      value: jenkins.jenkins.svc.cluster.local:50000
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/workspace
  volumes:
  - name: workspace-volume
    emptyDir: {}
"""
        }
    }

    environment {
        ECR_REGISTRY = "339007232055.dkr.ecr.us-east-1.amazonaws.com"
        IMAGE_REPO = "my-ecr"
        IMAGE_TAG = "1.${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                container('git') {
                    git branch: 'main', url: 'https://github.com/abd0hrz/gitops-cicd-pipeline-aws.git'
                }
            }
        }

        stage('Check Commit Author') {
            steps {
            container('git') {
                script {
                    
                    // Ensure Git works with Jenkins' workspace // Mark the current working directory as safe
                    sh 'git config --global --add safe.directory $(pwd)'

                    // Get and print latest commit author name and email
                    def commitAuthor = sh(script: 'git log -1 --pretty=format:"%an <%ae>"', returnStdout: true).trim()
                    echo "Latest commit author: ${commitAuthor}"

                    // Get the commit message instead of the author email
                    def commitMessage = sh(script: "git log -1 --pretty=%B", returnStdout: true).trim()
                    def firstLine = commitMessage.readLines()[0].trim()
                    echo "First line of commit message: ${firstLine}"

                    if (firstLine.contains("build: automatic update of nodeapp")) {
                        echo "Aborting pipeline after ArgoCD auto-commit"
                        currentBuild.result = 'ABORTED'
                        error('Pipeline aborted due to ArgoCD auto-commit')
                    }
            }
        }
    }
}

        stage('Build and Push Image') {
            steps {
                container('kaniko') {
                    script {
                        sh """
                            /kaniko/executor \\
                                --dockerfile=Applications/Dockerfile \\
                                --context=Applications \\
                                --destination=${ECR_REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}\\
                                --cache=true \\
                                --cache-ttl=24h
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Build completed"
        }
        success {
            echo "Image pushed successfully to ${ECR_REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo "Build failed"
        }
    }
}
