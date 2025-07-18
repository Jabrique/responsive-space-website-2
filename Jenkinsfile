def deployToEnv(String environmentName, String kubeconfigId, String namespace, String fullImageName) {
    echo "🚀 Deploying to ${environmentName} cluster..."
    withKubeConfig(credentialsId: kubeconfigId) {
        def configMapContent = """
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: cluster-info-cm
          namespace: ${namespace}
        data:
          cluster-info.txt: "Served from: ${environmentName}"
        """
        
        writeFile(file: 'configmap.yaml', text: configMapContent)

        sh """
        kubectl apply -f configmap.yaml
        
        sed -i 's|image: .*|image: ${fullImageName}|g' kubernetes/deployment.yml
        kubectl apply -f kubernetes/deployment.yml -n ${namespace}
        kubectl rollout status deployment/space-website -n ${namespace} --timeout=180s
        """
    }
}

pipeline {
    agent {
        kubernetes {
            cloud 'kubernetes'
            yaml """
            apiVersion: v1
            kind: Pod
            metadata:
              labels:
                jenkins-agent: kaniko-builder
            spec:
              containers:
              - name: kaniko
                image: gcr.io/kaniko-project/executor:debug
                command: ['sleep']
                args: ['infinity']
                tty: true
              - name: kubectl
                image: lachlanevenson/k8s-kubectl:latest
                command: ['sleep']
                args: ['infinity']
            """
        }
    }

    stages {
        stage('Build & Push Image') {
            steps {
                container('kaniko') {
                    script {
                        def dockerRegistry = "registry.cloudraya.com"
                        def imageName = "ir-cr-wowrack-781/space-website"
                        def fullImageName = "${dockerRegistry}/${imageName}:v${env.BUILD_NUMBER}"
                        
                        env.FULL_IMAGE_NAME = fullImageName

                        withCredentials([usernamePassword(credentialsId: 'cloudraya-container-registry', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                            echo "🔐 Creating Kaniko config.json..."
                            withEnv(["DOCKER_REGISTRY_HOST=${dockerRegistry}"]) {
                                sh '''
                                AUTH=$(echo -n "$REG_USER:$REG_PASS" | base64)
                                
                                echo '{ "auths": { "'$DOCKER_REGISTRY_HOST'": { "auth": "'$AUTH'" } } }' > /kaniko/.docker/config.json
                                '''
                            }
                        }
                        
                        echo "🔨 Building and pushing image: ${fullImageName}"
                        sh "/kaniko/executor --context `pwd` --dockerfile `pwd`/Dockerfile --destination ${fullImageName}"
                    }
                }
            }
        }
        
        stage('Deploy to ON-PREM') {
            steps {
                container('kubectl') {
                    script {
                        deployToEnv('ON-PREM', 'kubeconfig-onprem', 'demo', env.FULL_IMAGE_NAME)
                    }
                }
            }
        }

        stage('Approve for CLOUDRAYA') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Deployment ke On-Prem berhasil. Lanjutkan ke Cloudraya?', ok: 'Ya, Lanjutkan'
                }
            }
        }
        
        stage('Deploy to CLOUDRAYA') {
            steps {
                container('kubectl') {
                    script {
                        deployToEnv('CLOUDRAYA', 'kubeconfig-cloudraya', 'demo', env.FULL_IMAGE_NAME)
                    }
                }
            }
        }

        stage('Approve for AWS') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Deployment ke Cloudraya berhasil. Lanjutkan ke AWS?', ok: 'Ya, Lanjutkan'
                }
            }
        }

        stage('Deploy to AWS') {
            steps {
                container('kubectl') {
                    script {
                        deployToEnv('AWS', 'kubeconfig-aws', 'demo', env.FULL_IMAGE_NAME)
                    }
                }
            }
        }
    }
}
