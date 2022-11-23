# Sign with CodeSignTool

[![GitHub Actions Docker Status](https://github.com/bayrakmustafa/codesigner-docker/workflows/Docker%20Image%20CI/badge.svg)](https://github.com/bayrakmustafa/codesigner-docker)
[![GitlabCI](https://gitlab.com/mustafabayrak/codesigner-samples/badges/main/pipeline.svg?key_text=Gitlab%20CI)](https://gitlab.com/mustafabayrak/codesigner-samples/-/commits/main)

CodeSignTool is a secure, privacy-oriented multi-platform Java command line utility for remotely signing Microsoft Authenticode and Java code objects with eSigner EV code signing certificates. Hashes of the files are sent to SSL.com for signing so that the code itself is not sent. This is ideal where sensitive files need to be signed, but should not be sent over the wire for signing. CodeSignTool is also ideal for automated batch processes for high volume signings or integration into existing CI/CD pipeline workflows.

Docker image is used for code signing with GitlabCI. Detailed information: https://github.com/bayrakmustafa/codesigner-docker

# Usage (Job)

<!-- start usage -->
```yaml
# Below is the definition of your job to sign dll artifact
sign-dotnet-artifacts:
  # Define what stage the job will run in.
  stage: sign
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: docker:19.03.0
  # Similar to `image` property, but will link the specified services to the `image` container.
  services:
    - docker:19.03.0-dind
  # Defines environment variables for specific jobs.
  variables:
    COMMAND: "sign"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
  # Shell scripts executed by the Runner. Sign dotnet dll artifact with CodeSignTool Docker Image
  script:
    - docker pull ghcr.io/sslcom/codesigner:latest
    - docker run -i --rm --dns 8.8.8.8 --network host --volume ${CI_PROJECT_DIR}/packages:/codesign/examples --volume ${CI_PROJECT_DIR}/artifacts:/codesign/output 
        -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} 
        -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} 
        -input_file_path=/codesign/examples/${PROJECT_NAME}.dll -output_dir_path=/codesign/output
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/artifacts/${PROJECT_NAME}.dll
    expire_in: 1 days
  # Specify a list of job names from earlier stages from which artifacts should be loaded
  dependencies:
    - build-dotnet
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
# Groups jobs into stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

# Defines environment variables globally. Job level property overrides global variables
variables:
  PROJECT_NAME: "HelloWorld"
  PROJECT_VERSION: "0.0.1"
  DOTNET_VERSION: "3.1"
  ENVIRONMENT_NAME: "TEST"

# Below is the definition of your job to build dll artifact
build-dotnet:
  # Define what stage the job will run in.
  stage: build
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: mcr.microsoft.com/dotnet/sdk:3.1-bullseye
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
  # Shell scripts executed by the Runner. Build DLL artifact
  script:
    - dotnet build dotnet/${PROJECT_NAME}.csproj -c Release
    - cp dotnet/bin/Release/netcoreapp${DOTNET_VERSION}/${PROJECT_NAME}-${PROJECT_VERSION}.dll ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.dll
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/packages/HelloWorld.dll
    expire_in: 5 minutes

# Below is the definition of your job to sign dll artifact
sign-dotnet-artifacts:
  # Define what stage the job will run in.
  stage: sign
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: docker:19.03.0
  # Similar to `image` property, but will link the specified services to the `image` container.
  services:
    - docker:19.03.0-dind
  # Defines environment variables for specific jobs.
  variables:
    COMMAND: "sign"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
  # Shell scripts executed by the Runner. Sign dotnet dll artifact with CodeSignTool Docker Image
  script:
    - docker pull ghcr.io/sslcom/codesigner:latest
    - docker run -i --rm --dns 8.8.8.8 --network host --volume ${CI_PROJECT_DIR}/packages:/codesign/examples --volume ${CI_PROJECT_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}.dll -output_dir_path=/codesign/output
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/artifacts/${PROJECT_NAME}.dll
    expire_in: 1 days
  # Specify a list of job names from earlier stages from which artifacts should be loaded
  dependencies:
    - build-dotnet
```
<!-- end dotnet usage -->

## Java Code (Maven) JAR Signing Example Workflow

<!-- start maven usage -->
```yml
# Groups jobs into stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

# Defines environment variables globally. Job level property overrides global variables
variables:
  PROJECT_NAME: "HelloWorld"
  PROJECT_VERSION: "0.0.1"
  ENVIRONMENT_NAME: "TEST"

# Below is the definition of your job to build and sign jar artifact with Maven
build-maven-jar:
  # Define what stage the job will run in.
  stage: build
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: maven:3.8.3-openjdk-17
  # Create an environment variable
  variables:
    MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
    MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
  # Shell scripts executed by the Runner. Build JAR artifact on Maven
  script:
    - mvn $MAVEN_CLI_OPTS clean install -f java/pom.xml
    - cp java/target/${PROJECT_NAME}-${PROJECT_VERSION}.jar ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.jar
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.jar
    expire_in: 5 minutes

# Below is the definition of your job to sign maven jar artifacts
sign-maven-artifacts:
  # Define what stage the job will run in.
  stage: sign
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: docker:19.03.0
  # Similar to `image` property, but will link the specified services to the `image` container.
  services:
    - docker:19.03.0-dind
  # Defines environment variables for specific jobs.
  variables:
    COMMAND: "sign"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
    - cp ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.jar ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}-Maven.jar
  # Shell scripts executed by the Runner. Sign dotnet jar artifact with CodeSignTool Docker Image
  script:
    - docker pull ghcr.io/sslcom/codesigner:latest
    - docker run -i --rm --dns 8.8.8.8 --network host --volume ${CI_PROJECT_DIR}/packages:/codesign/examples --volume ${CI_PROJECT_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}-Maven.jar -output_dir_path=/codesign/output
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/artifacts/${PROJECT_NAME}-Maven.jar
    expire_in: 1 days
  # Specify a list of job names from earlier stages from which artifacts should be loaded
  dependencies:
    - build-maven-jar
```
<!-- end maven usage -->

## Java Code (Gradle) JAR Signing Example Workflow

<!-- start gradle usage -->
```yml
# Groups jobs into stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

# Defines environment variables globally. Job level property overrides global variables
variables:
  PROJECT_NAME: "HelloWorld"
  PROJECT_VERSION: "0.0.1"
  ENVIRONMENT_NAME: "TEST"

# Below is the definition of your job to build and sign jar artifact with Gradle
build-gradle-jar:
  # Define what stage the job will run in.
  stage: build
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: gradle:alpine
  # Defines environment variables for specific jobs.
  variables:
    GRADLE_OPTS: "-Dorg.gradle.daemon=false"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
    - GRADLE_USER_HOME="$(pwd)/.gradle"
    - export GRADLE_USER_HOME
  # Shell scripts executed by the Runner. Build JAR artifact on Gradle
  script:
    - gradle clean build -p java -PsetupType=jar
    - cp java/build/libs/${PROJECT_NAME}-${PROJECT_VERSION}.jar ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.jar
  # Used to specify a list of files and directories that should be attached to the job if it succeeds
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.jar
    expire_in: 5 minutes

# Below is the definition of your job to sign gradle jar artifacts
sign-gradle-artifacts:
  # Define what stage the job will run in.
  stage: sign
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: docker:19.03.0
  # Similar to `image` property, but will link the specified services to the `image` container.
  services:
    - docker:19.03.0-dind
  # Defines environment variables for specific jobs.
  variables:
    COMMAND: "sign"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
    - cp ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.jar ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}-Gradle.jar
  # Shell scripts executed by the Runner. Sign dotnet jar artifact with CodeSignTool Docker Image
  script:
    - docker pull ghcr.io/sslcom/codesigner:latest
    - docker run -i --rm --dns 8.8.8.8 --network host --volume ${CI_PROJECT_DIR}/packages:/codesign/examples --volume ${CI_PROJECT_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}-Gradle.jar -output_dir_path=/codesign/output
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/artifacts/${PROJECT_NAME}-Gradle.jar
    expire_in: 1 days
  # Specify a list of job names from earlier stages from which artifacts should be loaded
  dependencies:
    - build-gradle-jar
```
<!-- end gradle usage -->

## Java Code (Gradle) MSI Signing Example Workflow

<!-- start gradle usage -->
```yml
# Windows Runner
.windows_runners:
  # Used to select runners from the list of available runners. A runner must have all tags listed here to run the job.
  tags:
    - shared-windows
    - windows
    - windows-1809

stages:
  - build
  - sign

# Defines environment variables globally. Job level property overrides global variables
variables:
  PROJECT_NAME: "HelloWorld"
  PROJECT_VERSION: "0.0.1"
  ENVIRONMENT_NAME: "TEST"

# Below is the definition of your job to build msi artifact with Gradle
build-gradle-msi:
  # Define what stage the job will run in.
  stage: build
  # Runner for MSI Build on Windows
  extends:
    - .windows_runners
  # Defines environment variables for specific jobs.
  variables:
    JVM_OPTS: -Xmx3200m
    TERM: dumb
    GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=8"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
    - powershell.exe -ExecutionPolicy Bypass Install-WindowsFeature Net-Framework-Core
    - choco feature enable -n allowGlobalConfirmation
    - choco install dotnet3.5 --version=3.5.20160716 -y
    - choco install oraclejdk --version=17.0.2 -y
    - choco install gradle --version=7.4 -y
    - choco install wixtoolset --version=3.11.2 -y
    - New-Item ${CI_PROJECT_DIR}/java/gradle.properties
    - Set-Content ${CI_PROJECT_DIR}/java/gradle.properties 'org.gradle.java.home=C:/Program Files/Java/jdk-17.0.2'
  # Shell scripts executed by the Runner. Build MSI artifact on Windows
  script:
    - gradle build jpackage -x test --warning-mode all -p java -PsetupType=msi
    - Copy-Item "java/build/release/windows/*.msi" -Destination "${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.msi"
  # Used to specify a list of files and directories that should be attached to the job if it succeeds
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.msi
    expire_in: 5 minutes

# Below is the definition of your job to sign msi artifacts
sign-gradle-msi-artifacts:
  # Define what stage the job will run in.
  stage: sign
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: docker:19.03.0
  # Similar to `image` property, but will link the specified services to the `image` container.
  services:
    - docker:19.03.0-dind
  # Defines environment variables for specific jobs.
  variables:
    COMMAND: "sign"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
  # Shell scripts executed by the Runner. Sign dotnet msi artifact with CodeSignTool Docker Image
  script:
    - docker pull ghcr.io/sslcom/codesigner:latest
    - docker run -i --rm --dns 8.8.8.8 --network host --volume ${CI_PROJECT_DIR}/packages:/codesign/examples --volume ${CI_PROJECT_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}.msi -output_dir_path=/codesign/output
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/artifacts/${PROJECT_NAME}.msi
    expire_in: 1 days
  # Specify a list of job names from earlier stages from which artifacts should be loaded
  dependencies:
    - build-gradle-msi
```
<!-- end gradle usage -->

## Java Code (Gradle) EXE Signing Example Workflow

<!-- start gradle usage -->
```yml
# Windows Runner
.windows_runners:
  # Used to select runners from the list of available runners. A runner must have all tags listed here to run the job.
  tags:
    - shared-windows
    - windows
    - windows-1809

stages:
  - build
  - sign

# Defines environment variables globally. Job level property overrides global variables
variables:
  PROJECT_NAME: "HelloWorld"
  PROJECT_VERSION: "0.0.1"
  ENVIRONMENT_NAME: "TEST"

# Below is the definition of your job to build exe artifact with Gradle
build-gradle-exe:
  # Define what stage the job will run in.
  stage: build
  # Runner for MSI Build on Windows
  extends:
    - .windows_runners
  # Defines environment variables for specific jobs.
  variables:
    JVM_OPTS: -Xmx3200m
    TERM: dumb
    GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=8"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
    - powershell.exe -ExecutionPolicy Bypass Install-WindowsFeature Net-Framework-Core
    - choco feature enable -n allowGlobalConfirmation
    - choco install dotnet3.5 --version=3.5.20160716 -y
    - choco install oraclejdk --version=17.0.2 -y
    - choco install gradle --version=7.4 -y
    - choco install wixtoolset --version=3.11.2 -y
    - New-Item ${CI_PROJECT_DIR}/java/gradle.properties
    - Set-Content ${CI_PROJECT_DIR}/java/gradle.properties 'org.gradle.java.home=C:/Program Files/Java/jdk-17.0.2'
  # Shell scripts executed by the Runner. Build EXE artifact on Windows
  script:
    - gradle build jpackage -x test --warning-mode all -p java -PsetupType=exe
    - Copy-Item "java/build/release/windows/*.exe" -Destination "${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.exe"
  # Used to specify a list of files and directories that should be attached to the job if it succeeds
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.exe
    expire_in: 5 minutes

# Below is the definition of your job to sign exe artifacts
sign-gradle-exe-artifacts:
  # Define what stage the job will run in.
  stage: sign
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: docker:19.03.0
  # Similar to `image` property, but will link the specified services to the `image` container.
  services:
    - docker:19.03.0-dind
  # Defines environment variables for specific jobs.
  variables:
    COMMAND: "sign"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
  # Shell scripts executed by the Runner. Sign dotnet exe artifact with CodeSignTool Docker Image
  script:
    - docker pull ghcr.io/sslcom/codesigner:latest
    - docker run -i --rm --dns 8.8.8.8 --network host --volume ${CI_PROJECT_DIR}/packages:/codesign/examples --volume ${CI_PROJECT_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}.exe -output_dir_path=/codesign/output
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/artifacts/${PROJECT_NAME}.exe
    expire_in: 1 days
  # Specify a list of job names from earlier stages from which artifacts should be loaded
  dependencies:
    - build-gradle-exe
```
<!-- end gradle usage -->

## Powershell PS1 Signing Example Workflow

<!-- start powershell usage -->
```yml
stages:
  - build
  - sign

# Defines environment variables globally. Job level property overrides global variables
variables:
  PROJECT_NAME: "HelloWorld"
  PROJECT_VERSION: "0.0.1"
  ENVIRONMENT_NAME: "TEST"

# Below is the definition of your job to sign ps1 artifacts
sign-ps1-artifacts:
  # Define what stage the job will run in.
  stage: sign
  # Full name of the image that should be used. It should contain the Registry part if needed.
  image: docker:19.03.0
  # Similar to `image` property, but will link the specified services to the `image` container.
  services:
    - docker:19.03.0-dind
  # Defines environment variables for specific jobs.
  variables:
    COMMAND: "sign"
  # Defines scripts that should run *before* the job. Can be set globally or per job.
  before_script:
    - mkdir -p ${CI_PROJECT_DIR}/artifacts
    - mkdir -p ${CI_PROJECT_DIR}/packages
    - cp powershell/${PROJECT_NAME}.ps1 ${CI_PROJECT_DIR}/packages/${PROJECT_NAME}.ps1
  # Shell scripts executed by the Runner. Sign dotnet ps1 artifact with CodeSignTool Docker Image
  script:
    - docker pull ghcr.io/sslcom/codesigner:latest
    - docker run -i --rm --dns 8.8.8.8 --network host --volume ${CI_PROJECT_DIR}/packages:/codesign/examples --volume ${CI_PROJECT_DIR}/artifacts:/codesign/output -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} -input_file_path=/codesign/examples/${PROJECT_NAME}.ps1 -output_dir_path=/codesign/output
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  artifacts:
    paths:
      - ${CI_PROJECT_DIR}/artifacts/${PROJECT_NAME}.ps1
    expire_in: 1 days
```
<!-- end powershell usage -->

# CodeSignTool Guide

* https://www.ssl.com/guide/esigner-codesigntool-command-guide
* https://www.ssl.com/guide/remote-ev-code-signing-with-esigner/
