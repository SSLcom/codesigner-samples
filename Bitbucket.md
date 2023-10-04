# Sign with CodeSignTool

[![GitHub Actions Docker Status](https://github.com/sslcom/ci-images/workflows/Docker%20Image%20CI/badge.svg)](https://github.com/sslcom/ci-images)

CodeSignTool is a secure, privacy-oriented multi-platform Java command line utility for remotely signing Microsoft Authenticode and Java code objects with eSigner EV code signing certificates. Hashes of the files are sent to SSL.com for signing so that the code itself is not sent. This is ideal where sensitive files need to be signed, but should not be sent over the wire for signing. CodeSignTool is also ideal for automated batch processes for high volume signings or integration into existing CI/CD pipeline workflows.

Docker image is used for code signing with Bitbucket. Detailed information: https://github.com/sslcom/ci-images

# Usage (Job)

<!-- start usage -->
```yaml
- step:
    name: sign-dotnet-artifacts
    services:
      - docker
    script:
      # Created directories for artifacts
      - mkdir -p ${BITBUCKET_CLONE_DIR}/artifacts
      - mkdir -p ${BITBUCKET_CLONE_DIR}/packages
      # Fixed dotnet permission issue
      - chmod -R 777 ${BITBUCKET_CLONE_DIR}/packages
      # Docker Pull CodeSigner Docker Image
      - docker pull ghcr.io/sslcom/codesigner:latest
      # Sign artifact with CodeSigner docker image
      - docker run -i --rm --dns 8.8.8.8 --volume ${BITBUCKET_CLONE_DIR}/packages:/codesign/examples --volume ${BITBUCKET_CLONE_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.dll -output_dir_path=/codesign/output
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
```yml
pipelines:
  default:
    - step:
        # You can add a name to a step to make displays and reports easier to read and understand.
        name: build-dotnet
        # Name of the Docker image which may or may not include registry URL, tag, and digest value
        image: mcr.microsoft.com/dotnet/sdk:3.1-bullseye
        # Commands to execute in the step
        script:
          # Created directories for artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/packages
          # Build dotnet project with Release configuration
          - dotnet build dotnet/HelloWorld.csproj -c Release
          # Copy built artifacts to artifacts directory
          - cp dotnet/bin/Release/netcoreapp3.1/HelloWorld-0.0.1.dll ${BITBUCKET_CLONE_DIR}/packages/HelloWorld.dll
        # Files produced by a step to share with a following step
        artifacts:
          - packages/HelloWorld.dll
    - step:
        # You can add a name to a step to make displays and reports easier to read and understand.
        name: sign-dotnet-artifacts
        # Services enabled for the step
        services:
          - docker
        # Commands to execute in the step
        script:
          # Created directories for artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/packages
          # Fixed dotnet permission issue
          - chmod -R 777 ${BITBUCKET_CLONE_DIR}/packages
          # Docker Pull CodeSigner Docker Image
          - docker pull ghcr.io/sslcom/codesigner:latest
          # Sign artifact with CodeSigner docker image
          - docker run -i --rm --dns 8.8.8.8 --volume ${BITBUCKET_CLONE_DIR}/packages:/codesign/examples --volume ${BITBUCKET_CLONE_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.dll -output_dir_path=/codesign/output
```
<!-- end dotnet usage -->

## Java Code (Maven) JAR Signing Example Workflow

<!-- start maven usage -->
```yml
pipelines:
  default:
    - step:
        # You can add a name to a step to make displays and reports easier to read and understand.
        name: build-maven-java
        # Name of the Docker image which may or may not include registry URL, tag, and digest value
        image: maven:3.6.3
        # Caches enabled for the step
        caches:
          - maven
        # Commands to execute in the step
        script:
          # Created directories for artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/packages
          # Build Maven project with Maven Options
          - mvn --batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true clean install -f java/pom.xml
          # Copy built artifacts to artifacts directory
          - cp java/target/HelloWorld-0.0.1.jar ${BITBUCKET_CLONE_DIR}/packages/HelloWorld-Maven.jar
        # Files produced by a step to share with a following step
        artifacts:
          - packages/HelloWorld-Maven.jar
    - step:
        # You can add a name to a step to make displays and reports easier to read and understand.
        name: sign-maven-artifacts
        # Services enabled for the step
        services:
          - docker
        # Commands to execute in the step
        script:
          # Created directories for artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/packages
          # Docker Pull CodeSigner Docker Image
          - docker pull ghcr.io/sslcom/codesigner:latest
          # Sign artifact with CodeSigner docker image
          - docker run -i --rm --dns 8.8.8.8 --volume ${BITBUCKET_CLONE_DIR}/packages:/codesign/examples --volume ${BITBUCKET_CLONE_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld-Maven.jar -output_dir_path=/codesign/output
```
<!-- end maven usage -->

## Java Code (Gradle) JAR Signing Example Workflow

<!-- start gradle usage -->
```yml
pipelines:
  default:
    - step:
        # You can add a name to a step to make displays and reports easier to read and understand.
        name: build-gradle-java
        # Name of the Docker image which may or may not include registry URL, tag, and digest value
        image: gradle:7.2-jdk11
        # Caches enabled for the step
        caches:
          - gradle
        # Commands to execute in the step
        script:
          # Created directories for artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/packages
          # Set GRADLE options for building Java project
          - export GRADLE_OPTS="-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=4"
          - touch ${BITBUCKET_CLONE_DIR}/java/gradle.properties
          - echo "${BITBUCKET_CLONE_DIR}/java/gradle.properties" >> 'org.gradle.daemon=false'
          - echo "${BITBUCKET_CLONE_DIR}/java/gradle.properties" >> 'org.gradle.workers.max=4'
          # Build Maven project with Maven Options
          - gradle clean build -p java -PsetupType=jar
          # Copy built artifacts to artifacts directory
          - cp java/build/libs/HelloWorld-0.0.1.jar ${BITBUCKET_CLONE_DIR}/packages/HelloWorld-Gradle.jar
        # Files produced by a step to share with a following step
        artifacts:
          - packages/HelloWorld-Gradle.jar
    - step:
        # You can add a name to a step to make displays and reports easier to read and understand.
        name: sign-gradle-artifacts
        # Services enabled for the step
        services:
          - docker
        # Commands to execute in the step
        script:
          # Created directories for artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/packages
          # Docker Pull CodeSigner Docker Image
          - docker pull ghcr.io/sslcom/codesigner:latest
          # Sign artifact with CodeSigner docker image
          - docker run -i --rm --dns 8.8.8.8 --volume ${BITBUCKET_CLONE_DIR}/packages:/codesign/examples --volume ${BITBUCKET_CLONE_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld-Gradle.jar -output_dir_path=/codesign/output
```
<!-- end gradle usage -->

## Powershell PS1 Signing Example Workflow

<!-- start powershell usage -->
```yml
pipelines:
  default:
    - step:
        # You can add a name to a step to make displays and reports easier to read and understand.
        name: sign-ps1-artifacts
        # Services enabled for the step
        services:
          - docker
        script:
          # Created directories for artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/artifacts
          - mkdir -p ${BITBUCKET_CLONE_DIR}/packages
          # Copy ps1 script for signing
          - cp powershell/HelloWorld.ps1 ${BITBUCKET_CLONE_DIR}/packages/HelloWorld.ps1
          # Docker Pull CodeSigner Docker Image
          - docker pull ghcr.io/sslcom/codesigner:latest
          # Sign artifact with CodeSigner docker image
          - docker run -i --rm --dns 8.8.8.8 --volume ${BITBUCKET_CLONE_DIR}/packages:/codesign/examples --volume ${BITBUCKET_CLONE_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.ps1 -output_dir_path=/codesign/output
```
<!-- end powershell usage -->

# CodeSignTool Guide

* https://www.ssl.com/guide/esigner-codesigntool-command-guide
* https://www.ssl.com/guide/remote-ev-code-signing-with-esigner/
