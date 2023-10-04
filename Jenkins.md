# Sign with CodeSignTool

[![GitHub Actions Docker Status](https://github.com/sslcom/ci-images/workflows/Docker%20Image%20CI/badge.svg)](https://github.com/sslcom/ci-images)
[![Build Status](https://img.shields.io/travis/sslcom/codesigner-samples.svg?style=plastic&label=Travis%20CI&branch=main)](https://app.travis-ci.com/sslcom/codesigner-samples)

CodeSignTool is a secure, privacy-oriented multi-platform Java command line utility for remotely signing Microsoft Authenticode and Java code objects with eSigner EV code signing certificates. Hashes of the files are sent to SSL.com for signing so that the code itself is not sent. This is ideal where sensitive files need to be signed, but should not be sent over the wire for signing. CodeSignTool is also ideal for automated batch processes for high volume signings or integration into existing CI/CD pipeline workflows.

Docker image is used for code signing with Jenkins. Detailed information: https://github.com/sslcom/ci-images

# Usage (Job)

<!-- start usage -->
```groovy
stage('Sign and Save Dotnet Core DLL Artifact') {
    steps {
        sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output 
            -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} 
            ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.dll -output_dir_path=/codesign/output'
    }
    post {
        always {
            archiveArtifacts artifacts: "artifacts/HelloWorld.dll", onlyIfSuccessful: true
        }
    }
}
```
<!-- end usage -->

### Environment Variables

* `USERNAME`: SSL.com account username. (**Required**)

* `PASSWORD`: SSL.com account password (**Required**)

* `CREDENTIAL_ID`: Credential ID for signing certificate. If credential_id is omitted and the user has only one eSigner code signing certificate, CodeSignTool will default to that. If the user has more than one code signing certificate, this parameter is mandatory. (**Required**)

* `TOTP_SECRET`: OAuth TOTP Secret. You can access detailed information on https://www.ssl.com/how-to/automate-esigner-ev-code-signing (**Required**)

* `ENVIRONMENT_NAME` : 'TEST' or 'PROD' Environment. (**Required**)

### Inputs

* `input_file_path`: Path of code object to be signed. (**Required**)

* `output_dir_path`: Directory where signed code object(s) will be written. If output_path is omitted, the file specified in -file_path will be overwritten with the signed file.

# Examples

## Dotnet Code DLL Signing Example Workflow

<!-- start dotnet usage -->
```groovy
pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: "5"))
        disableConcurrentBuilds()
    }

    // Install Build Tools
    tools {
        dotnetsdk "DOTNET_CORE_3.1.24"  //https://plugins.jenkins.io/dotnet-sdk
    }

    // Create an environment variable
    environment {
        USERNAME          = credentials('es-username')       // SSL.com account username.
        PASSWORD          = credentials('es-password')       // SSL.com account password.
        CREDENTIAL_ID     = credentials('es-crendential-id') // Credential ID for signing certificate.
        TOTP_SECRET       = credentials('es-totp-secret')    // OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
        
        REGISTRY_PASSWORD = credentials('es-github-token')   // Github Token for Pull codesigner Docker Image
        REGISTRY_USERNAME = 'sslcom'                  // Github Registry Username for Pull codesigner Docker Image
        ENVIRONMENT_NAME  = 'TEST'                           // SSL.com Environment Name. For Demo Account It can be 'TEST' otherwise it will be 'PROD'
    }

    stages {
        // 1) Create Artifact Directory for store signed and unsigned artifact files
        stage('Prepare for Signing') {
            steps {
                sh 'mkdir ${WORKSPACE}/artifacts'
                sh 'mkdir ${WORKSPACE}/packages'
            }
        }

        // 2) Pull Codesigner Docker Image From Github Registry
        stage('Docker Pull Image') {
            steps {
                sh 'docker pull ghcr.io/sslcom/codesigner:latest'
            }
        }

        // 3) Build a dotnet project or solution and all of its dependencies.
        //    After it has been created dll or exe file, copy to 'packages' folder for siging
        stage('Build Dotnet Core DLL') {
            steps {
                sh 'dotnet build dotnet/HelloWorld.csproj -c Release'
                sh 'cp dotnet/bin/Release/netcoreapp3.1/HelloWorld-0.0.1.dll ${WORKSPACE}/packages/HelloWorld.dll'
            }
        }

        // 4) This is the step where the created DLL (artifact) files will be signed with CodeSignTool.
        stage('Sign and Save Dotnet Core DLL Artifact') {
            steps {
                sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output 
                    -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} 
                    ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.dll -output_dir_path=/codesign/output'
            }
            post {
                always {
                    archiveArtifacts artifacts: "artifacts/HelloWorld.dll", onlyIfSuccessful: true
                }
            }
        }
    }
}
```
<!-- end dotnet usage -->

## Java Code (Maven) JAR Signing Example Workflow

<!-- start maven usage -->
```groovy
pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: "5"))
        disableConcurrentBuilds()
    }

    // Install Build Tools
    tools {
        maven "MAVEN_3.8.5"             //https://plugins.jenkins.io/maven-plugin
        jdk "JDK_9.0.4"                 //https://plugins.jenkins.io/jdk-tool
    }

    // Create an environment variable
    environment {
        USERNAME          = credentials('es-username')       // SSL.com account username.
        PASSWORD          = credentials('es-password')       // SSL.com account password.
        CREDENTIAL_ID     = credentials('es-crendential-id') // Credential ID for signing certificate.
        TOTP_SECRET       = credentials('es-totp-secret')    // OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
        
        REGISTRY_PASSWORD = credentials('es-github-token')   // Github Token for Pull codesigner Docker Image
        REGISTRY_USERNAME = 'sslcom'                  // Github Registry Username for Pull codesigner Docker Image
        ENVIRONMENT_NAME  = 'TEST'                           // SSL.com Environment Name. For Demo Account It can be 'TEST' otherwise it will be 'PROD'
    }

    stages {
        // 1) Create Artifact Directory for store signed and unsigned artifact files
        stage('Prepare for Signing') {
            steps {
                sh 'mkdir ${WORKSPACE}/artifacts'
                sh 'mkdir ${WORKSPACE}/packages'
            }
        }

        // 2) Pull Codesigner Docker Image From Github Registry
        stage('Docker Pull Image') {
            steps {
                sh 'docker pull ghcr.io/sslcom/codesigner:latest'
            }
        }

        // 3) Build a maven project or solution and all of its dependencies.
        //    After it has been created jar file, copy to 'packages' folder for siging
        stage('Build Maven JAR') {
            steps {
                sh 'mvn clean install -f java/pom.xml'
                sh 'cp java/target/HelloWorld-0.0.1.jar ${WORKSPACE}/packages/HelloWorld.jar'
            }
        }

        // 4) This is the step where the created JAR (artifact) files will be signed with CodeSignTool.
        stage('Sign and Save Maven JAR Artifact') {
            steps {
                sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output 
                    -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} 
                    ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.jar -output_dir_path=/codesign/output'
            }
            post {
                always {
                    archiveArtifacts artifacts: "artifacts/HelloWorld.jar", onlyIfSuccessful: true
                }
            }
        }
    }
}
```
<!-- end maven usage -->

## Java Code (Gradle) JAR Signing Example Workflow

<!-- start gradle usage -->
```groovy
pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: "5"))
        disableConcurrentBuilds()
    }

    // Install Build Tools
    tools {
        dotnetsdk "DOTNET_CORE_3.1.24"  //https://plugins.jenkins.io/dotnet-sdk
    }

    // Create an environment variable
    environment {
        USERNAME          = credentials('es-username')       // SSL.com account username.
        PASSWORD          = credentials('es-password')       // SSL.com account password.
        CREDENTIAL_ID     = credentials('es-crendential-id') // Credential ID for signing certificate.
        TOTP_SECRET       = credentials('es-totp-secret')    // OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
        
        REGISTRY_PASSWORD = credentials('es-github-token')   // Github Token for Pull codesigner Docker Image
        REGISTRY_USERNAME = 'sslcom'                  // Github Registry Username for Pull codesigner Docker Image
        ENVIRONMENT_NAME  = 'TEST'                           // SSL.com Environment Name. For Demo Account It can be 'TEST' otherwise it will be 'PROD'
    }

    stages {
        // 1) Create Artifact Directory for store signed and unsigned artifact files
        stage('Prepare for Signing') {
            steps {
                sh 'mkdir ${WORKSPACE}/artifacts'
                sh 'mkdir ${WORKSPACE}/packages'
            }
        }

        // 2) Pull Codesigner Docker Image From Github Registry
        stage('Docker Pull Image') {
            steps {
                sh 'docker pull ghcr.io/sslcom/codesigner:latest'
            }
        }

        // 3) Build a gradle project or solution and all of its dependencies.
        //    After it has been created jar file, copy to 'packages' folder for siging
        stage('Build Gradle JAR') {
            steps {
                sh 'gradle clean build -p java -PsetupType=jar'
                sh 'cp java/build/libs/HelloWorld-0.0.1.jar ${WORKSPACE}/packages/HelloWorld.jar'
            }
        }

        // 8) This is the step where the created JAR (artifact) files will be signed with CodeSignTool.
        stage('Sign and Save Gradle JAR Artifact') {
            steps {
                sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output 
                    -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} 
                    ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.jar -output_dir_path=/codesign/output'
            }
            post {
                always {
                    archiveArtifacts artifacts: "artifacts/HelloWorld.jar", onlyIfSuccessful: true
                }
            }
        }
    }
}
```
<!-- end gradle usage -->

## Powershell PS1 Signing Example Workflow

<!-- start powershell usage -->
```groovy
pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: "5"))
        disableConcurrentBuilds()
    }

    // Create an environment variable
    environment {
        USERNAME          = credentials('es-username')       // SSL.com account username.
        PASSWORD          = credentials('es-password')       // SSL.com account password.
        CREDENTIAL_ID     = credentials('es-crendential-id') // Credential ID for signing certificate.
        TOTP_SECRET       = credentials('es-totp-secret')    // OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
        
        REGISTRY_PASSWORD = credentials('es-github-token')   // Github Token for Pull codesigner Docker Image
        REGISTRY_USERNAME = 'sslcom'                  // Github Registry Username for Pull codesigner Docker Image
        ENVIRONMENT_NAME  = 'TEST'                           // SSL.com Environment Name. For Demo Account It can be 'TEST' otherwise it will be 'PROD'
    }

    stages {
        // 1) Create Artifact Directory for store signed and unsigned artifact files
        stage('Prepare for Signing') {
            steps {
                sh 'mkdir ${WORKSPACE}/artifacts'
                sh 'mkdir ${WORKSPACE}/packages'
            }
        }

        // 2) Pull Codesigner Docker Image From Github Registry
        stage('Docker Pull Image') {
            steps {
                sh 'docker pull ghcr.io/sslcom/codesigner:latest'
            }
        }

        // 3) This is the step PS1 file will be signed with CodeSignTool.
        stage('Sign and Save PS1 Artifact') {
            steps {
                sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output 
                    -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} 
                    ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.ps1 -output_dir_path=/codesign/output'
            }
            post {
                always {
                    archiveArtifacts artifacts: "artifacts/HelloWorld.ps1", onlyIfSuccessful: true
                }
            }
        }
    }
}
```
<!-- end powershell usage -->

# CodeSignTool Guide

* https://www.ssl.com/guide/esigner-codesigntool-command-guide
* https://www.ssl.com/guide/remote-ev-code-signing-with-esigner/
