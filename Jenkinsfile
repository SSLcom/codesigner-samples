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
        gradle "GRADLE_7.4.2"           //https://plugins.jenkins.io/gradle
        dotnetsdk "DOTNET_CORE_3.1.24"  //https://plugins.jenkins.io/dotnet-sdk
    }

    // Create an environment variable
    environment {
        USERNAME          = credentials('es-username')       // SSL.com account username.
        PASSWORD          = credentials('es-password')       // SSL.com account password.
        CREDENTIAL_ID     = credentials('es-crendential-id') // Credential ID for signing certificate.
        TOTP_SECRET       = credentials('es-totp-secret')    // OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
        
        REGISTRY_PASSWORD = credentials('es-github-token')   // Github Token for Pull codesigner Docker Image
        REGISTRY_USERNAME = 'bayrakmustafa'                  // Github Registry Username for Pull codesigner Docker Image
        ENVIRONMENT_NAME  = 'TEST'                           // SSL.com Environment Name. For Demo Account It can be 'TEST' otherwise it will be 'PROD'
        // CodeSignTool Commands:
        // - get_credential_ids: Output the list of eSigner credential IDs associated with a particular user.
        // - credential_info: Output key and certificate information related to a credential ID.
        // - sign: Sign and timestamp code object.
        // - batch_sign: Sign and timestamp multiple code objects with one OTP.
        // - hash: Pre-compute hash(es) for later use with batch_hash_sign command.
        // - batch_sign_hash: Sign hash(es) pre-computed with hash command.
        COMMAND          = 'sign'

        PROJECT_NAME     = 'HelloWorld'
        PROJECT_VERSION  = '0.0.1'
        MAVEN_VERSION    = '3.8.5'
        JAVA_VERSION     = '9.0.4'
        DOTNET_VERSION   = '3.1'
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

        // 3) Build Projects (Maven, Gradle, Dotnet, Powershell)
        stage('Build Packages') {
            parallel {
                // 4) Copy Powershell script to packages directory for signing
                stage('Copy Powershell Script') {
                    steps {
                        sh 'cp powershell/${PROJECT_NAME}.ps1 ${WORKSPACE}/packages/${PROJECT_NAME}.ps1'
                    }
                }
                // 4) Build a dotnet project or solution and all of its dependencies.
                //    After it has been created dll or exe file, copy to 'packages' folder for siging
                stage('Build Dotnet Core DLL') {
                    steps {
                        sh 'dotnet build dotnet/${PROJECT_NAME}.csproj -c Release'
                        sh 'cp dotnet/bin/Release/netcoreapp${DOTNET_VERSION}/${PROJECT_NAME}-${PROJECT_VERSION}.dll ${WORKSPACE}/packages/${PROJECT_NAME}.dll'
                    }
                }
                // 4) Build a maven project or solution and all of its dependencies.
                //    After it has been created jar file, copy to 'packages' folder for siging
                stage('Build Maven JAR') {
                    steps {
                        sh 'mvn clean install -f java/pom.xml'
                        sh 'cp java/target/${PROJECT_NAME}-${PROJECT_VERSION}.jar ${WORKSPACE}/packages/${PROJECT_NAME}-Maven.jar'
                    }
                }
                // 4) Build a gradle project or solution and all of its dependencies.
                //    After it has been created jar file, copy to 'packages' folder for siging
                stage('Build Gradle JAR') {
                    steps {
                        sh 'gradle clean build -p java -PsetupType=jar'
                        sh 'cp java/build/libs/${PROJECT_NAME}-${PROJECT_VERSION}.jar ${WORKSPACE}/packages/${PROJECT_NAME}-Gradle.jar'
                    }
                }
            }
        }

        // 5) This is the step PS1 file will be signed with CodeSignTool.
        stage('Sign and Save PS1 Artifact') {
            steps {
                sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}.ps1 -output_dir_path=/codesign/output'
            }
            post {
                always {
                    archiveArtifacts artifacts: "artifacts/${PROJECT_NAME}.ps1", onlyIfSuccessful: true
                }
            }
        }

        // 6) This is the step where the created DLL (artifact) files will be signed with CodeSignTool.
        stage('Sign and Save Dotnet Core DLL Artifact') {
            steps {
                sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}.dll -output_dir_path=/codesign/output'
            }
            post {
                always {
                    archiveArtifacts artifacts: "artifacts/${PROJECT_NAME}.dll", onlyIfSuccessful: true
                }
            }
        }

        // 7) This is the step where the created JAR (artifact) files will be signed with CodeSignTool.
        stage('Sign and Save Maven JAR Artifact') {
            steps {
                sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}-Maven.jar -output_dir_path=/codesign/output'
            }
            post {
                always {
                    archiveArtifacts artifacts: "artifacts/${PROJECT_NAME}-Maven.jar", onlyIfSuccessful: true
                }
            }
        }
    
        // 8) This is the step where the created JAR (artifact) files will be signed with CodeSignTool.
        stage('Sign and Save Gradle JAR Artifact') {
            steps {
                sh 'docker run -i --rm --dns 8.8.8.8 --network host --volume ${WORKSPACE}/packages:/codesign/examples --volume ${WORKSPACE}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}-Gradle.jar -output_dir_path=/codesign/output'
            }
            post {
                always {
                    archiveArtifacts artifacts: "artifacts/${PROJECT_NAME}-Gradle.jar", onlyIfSuccessful: true
                }
            }
        }
    }
}
