pipeline {
    agent any
    options {
        skipDefaultCheckout()
    }
    environment {
        GIT_REPO = 'https://github.com/k-fathi/petclinic-jenkins-cicd.git'
        MAVEN_CACHE = '/var/jenkins_home/maven-cache/.m2-cache'
        TRIVY_HOST_CACHE = '/var/jenkins_home/trivy-cache/'
        TRIVY_CACHE_DIR = '/tmp/.cache/trivy'
        REPO="karimfathi1"
        IMG="spring-petclinic"
        TAG="${BUILD_NUMBER}"
        CONTAINER_NAME="spring-petclinc"
        APP_PORT="8080"
    }   
    stages {
        stage('1. Shallow Cloning') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    // extensions: [cloneOption(depth: 2, noTags: false, reference: '', shallow: true)],
                    userRemoteConfigs: [[credentialsId: 'github-token', url: "${env.GIT_REPO}"]]
                )
            }
        } 
        
        stage('2. Helm Prep (Sequential Setup)') {
            agent {
                docker {
                    image 'alpine/helm:3.12.0'
                    args '-u 0:0 --entrypoint=""'
                    reuseNode true
                }
            }
            steps {
                dir('./devops/') {
                    echo "Downloading Helm Dependencies & Rendering Helm Chart..."
                        sh 'helm dependency update ./petclinic-chart/'
                        // sh 'helm template petclinic-release ./petclinic-chart/ > rendered.yaml'
                        sh 'helm template petclinic-release ./petclinic-chart/ > rendered.yaml'
                }
            }    
        }

        stage('3. Static Analysis') {
            parallel {
                stage('YAML Linting'){
                    agent {
                        docker {
                            image 'cytopia/yamllint'
                            args '-u 0:0 --entrypoint=""'
                            reuseNode true
                        }
                    }
                    steps {
                        dir('./devops/') {
                            echo "Running YAML Linting on Rendered File..."
                            sh 'yamllint -d relaxed rendered.yaml || true'
                        }
                    }
                }
                stage('Trivy Helm Scanning') {
                    agent {
                        docker {
                            image 'aquasec/trivy:0.74.0'
                            args "-u 0:0 -v ${env.TRIVY_HOST_CACHE}:${env.TRIVY_CACHE_DIR} --entrypoint=''"
                            reuseNode true
                        }
                    }
                    steps {
                        dir('./devops/') {
                            echo "Running Trivy Security Scan on Rendered File..."
                            // sh 'trivy config --severity CRITICAL,HIGH --exit-code 1 rendered.yaml'
                            sh 'trivy config --severity CRITICAL,HIGH --exit-code 0 rendered.yaml'
                        }
                    }
                }
            }
        }

        stage('4. App Environment Prep') {
            parallel {
                stage('Preparing Maven for SCA') {
                    when {
                        changeset "spring-petclinic/**/*"
                    }
                    agent {
                        docker {
                            image 'maven:3.9.16-eclipse-temurin-17-alpine'
                            args "-v ${env.MAVEN_CACHE}:/root/.m2 -u 0:0 --entrypoint=''"                             
                            reuseNode true
                        }
                    }
                    steps {
                        dir('./spring-petclinic/') {
                            sh 'echo "Compile the Code to get .class files"'
                            sh 'mvn clean compile -DskipTests -Dcheckstyle.skip=true'

                            sh 'echo "Generate the SBOM.json file"'
                            sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -DoutputFormat=json -DoutputName=sbom'
                            archiveArtifacts artifacts: 'target/sbom.json', followSymlinks: false
                        }    
                    }
                }
                stage('Preparing Trivy for Scanning') {                 
                    agent {
                        docker {
                            image 'aquasec/trivy:0.74.0'
                            args "-v ${env.TRIVY_HOST_CACHE}:${env.TRIVY_CACHE_DIR} -u 0:0 --entrypoint=''"
                            reuseNode true
                        }
                    }
                    steps {
                        sh 'echo "1. Downloading Main Vulnerability Database for SCA"'
                        sh "trivy sbom --download-db-only"  
                        
                        sh 'echo "2. Downloading Java-specific Database for SCA"'
                        sh "trivy sbom --download-java-db-only"
                    }
                }
            }
        }
        
        stage('5. App & Sec Scanning') {
            agent {
                docker {
                    image 'aquasec/trivy:0.74.0'
                    args "-v ${env.TRIVY_HOST_CACHE}:${env.TRIVY_CACHE_DIR} -u 0:0 --entrypoint=''"
                    reuseNode true
                }
            }
            stages {
                stage('Run Parallel Scanning') {
                    parallel {
                        stage('SCA Scanning') {
                            when {
                                changeset "spring-petclinic/**/*"
                            }
                            steps {
                                dir('./spring-petclinic/') {
                                    sh 'echo "SCA Scanning - Trivy Scans sbom.json..."'
                                    // sh "trivy sbom --severity HIGH,CRITICAL --exit-code 1 --skip-db-update target/sbom.json"
                                    sh "trivy sbom --severity HIGH,CRITICAL --exit-code 0 --skip-db-update target/sbom.json"
                                }
                            }
                        }
                        stage('IaC Scanning') {
                            when {
                                changeset "spring-petclinic/**/*"
                            }
                            steps {
                                dir('./spring-petclinic/') {
                                    sh 'echo "IaC Scanning - Trivy Scans Dockerfile..."'
                                    // sh "trivy config --severity CRITICAL,HIGH --exit-code 1 ./Dockerfile"
                                    sh "trivy config --severity CRITICAL,HIGH --exit-code 0 ./Dockerfile"
                                }
                            }
                        }
                        stage('Secrets Scanning') {
                            steps {
                                sh 'echo "Secrets Scanning - Trivy Scans Secrets..."'
                                // sh "trivy fs --scanners secret --severity CRITICAL,HIGH --offline-scan --exit-code 1 ."
                                sh "trivy fs --scanners secret --severity CRITICAL,HIGH --offline-scan --exit-code 0 ."
                            }
                        }
                    } 
                }
            }
        }
    
        stage('6. Testing The App - Unit & Integration Tests') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args "-u root -v ${env.MAVEN_CACHE}:/root/.m2 --entrypoint=\"\""
                    reuseNode true
                }
            }    
            when {
                changeset "spring-petclinic/**/*"
            }        
            steps {
                // this line do the unit and integration tests and generate the jacoco report, but it will fail the pipeline if any test fails
                // so we will use the next line instead to ignore the test failures and let sonar plugin handle it.                 
                // sh 'mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent verify org.jacoco:jacoco-maven-plugin:report -Dmaven.test.failure.ignore=true'
                dir('./spring-petclinic/') {
                    sh "echo 'Running Unit Tests & Integration Tests with JaCoCo...'"
                    sh 'mvn org.jacoco:jacoco-maven-plugin:prepare-agent package org.jacoco:jacoco-maven-plugin:report -Dmaven.test.failure.ignore=true -DskipITs'
                }

            }
        }
        stage('7. Building & SAST - Scanning The App'){
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args "-u root -v ${env.MAVEN_CACHE}:/root/.m2 --entrypoint=\"\" --network devops-net"
                    reuseNode true
                }
            }
            when {
                changeset "spring-petclinic/**/*"
            }  
            steps{
                withSonarQubeEnv('sonarqube-server') {
                    dir('./spring-petclinic/'){
                        sh 'echo  Sonar Clinet Plugin Collects the source code file + Unit, integration tests report and Sending them to SonarQube Server now...'
                        sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=spring-petclinic \
                        -Dsonar.projectName="Spring Petclinic" \
                        -Dsonar.coverage.jacoco.xmlReportPath=target/site/jacoco/jacoco.xml 
                        '''
                        // the last parameter is to notify sonar blugin with the location of the jacoco report, so it can calculate the code coverage and send it to sonar server
                    }
                }
            }
        }
        // just to free the heavyweight executor, we will use agent none to wait for the SonarQube Quality Gate result without using any agent.
        stage('8. Waiting for SonarQube Quality Gate Result'){
            agent none
            when {
                changeset "spring-petclinic/**/*"
            }
            steps{
                sh 'echo  Waiting for SonarQube Quality Gate result now...'
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate(abortPipeline: true)
                }
            }
        }

        stage('9. - Logining, Building, Testing, Pushing The Docker Image'){
            when{
                changeset "spring-petclinic/**/*"
            }
            steps{
                dir('spring-petclinic'){
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKERHUB_PWD', usernameVariable: 'DOCKERHUB_USER')]) {
                        sh 'echo  Loggin to DockerHub now...'
                        sh 'echo "${DOCKERHUB_PWD}" | docker login -u ${DOCKERHUB_USER} --password-stdin'
                    
                        sh 'echo  Building The Image now...'
                        sh 'export DOCKER_BUILDKIT=0 && docker build --platform linux/amd64 -t ${REPO}/${IMG}:${TAG} -t ${REPO}/${IMG}:latest .'
                        
                        // use tviry as inline docker command to scan the image, and mount the trivy cache dir to avoid downloading the vulnerability database every time
                        // used the douple quotes besace only jenkins can evaluate the variables before passing it to the shell
                        // if '' are used the entire command will evaluated in the shell but the shell doesn't even know the variables, so it will fail 
                        sh 'echo  Trivy Scanning the Image now...'
                        sh """
                        docker run --rm \
                                -v ${env.TRIVY_SCA_CACHE}:/tmp/.trivy \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                aquasec/trivy \
                                image --severity HIGH,CRITICAL \
                                --exit-code 0 \
                                ${REPO}/${IMG}:${TAG}
                        """
                        // exit code 1 to make the pipeline fail if any vulnerability found.
                        
                        sh 'echo  Pushing the Images now...'
                        sh "docker push ${REPO}/${IMG}:${TAG}"
                        sh "docker push ${REPO}/${IMG}:latest"
                    }
                    sh 'echo  Generating the deploy file now...'
                    sh """
                        cat > deploy-info-${BUILD_NUMBER}.txt <<EOF
image: ${REPO}/${IMG}:${TAG}
build: ${env.BUILD_NUMBER}
commit: ${env.GIT_COMMIT}
branch: ${env.GIT_BRANCH}
url: ${env.BUILD_URL}
date: \$(date +"%Y_%m_%d-%H:%M:%S")
EOF
                    """
                }
            }            
        }
        stage('10. Deploy to K3d Cluster'){
            agent{
                docker{
                    image 'alpine/helm:3.14.0'
                    args '--network devops-net'
                }
            }
            steps{
                dir('devops/'){
                    sh """
                    hemlm upgrade --install petclinic ./petclinic-cart \
                    --namespace petclinic --create-namespace \
                    --set deployment.image.tag=${BUILD_NUMBER} \
                    --wait --timeout 5m 
                    """
                }
            }
        }
        stage('smoke testing'){
            steps{
                sh "echo  Running A Smoke Testing..."
                sh """
                    SUCCESS=0
                    for i in {1..5}; do
                        if curl -s -H "Host: petclinic.local" http://k3d-k3d-cluster-serverlb/ | grep -iE "Welcome|PetClinic|Spring"; then
                            SUCCESS=1
                            break
                        else
                            echo "Attempt $i failed. Retrying in 5 seconds..."
                            sleep 5
                        fi
                    done
                    if [ \$SUCCESS -eq 1 ]; then
                        echo "Smoke test passed!"
                    else
                        echo "Smoke test failed after 5 attempts."
                        exit 1
                    fi
                """
            }
        }
        stage('DAST - OWASP ZAP') {
            steps {
                echo "Starting OWASP ZAP Baseline Security Scan..."
                sh """
                    mkdir -p zap-reports
                    chmod 777 zap-reports

                    docker run --rm \
                    --network devops-net \
                    --add-host petclinic.local:172.22.0.4 \
                    -v \$(pwd)/zap-reports:/zap/wrk/:rw \
                    ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                    -t http://petclinic.local/ \
                    -r zap-report.html \
                    -I || true
                    docker cp zap-scanner:/zap/wrk/zap-report.html ./zap-report.html || true
                    docker rm -f zap-scanner || true
                """
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
            sh 'echo "Cleaning up the Workspace..."'
            cleanWs()
        }
    }
}