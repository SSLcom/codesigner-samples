# Sign with CodeSignTool

[![GitHub Actions Docker Status](https://github.com/sslcom/ci-images/workflows/Docker%20Image%20CI/badge.svg)](https://github.com/sslcom/ci-images)
[![Build Status](https://img.shields.io/travis/sslcom/codesigner-samples.svg?style=plastic&label=Travis%20CI&branch=main)](https://app.travis-ci.com/sslcom/codesigner-samples)

CodeSignTool is a secure, privacy-oriented multi-platform Java command line utility for remotely signing Microsoft Authenticode and Java code objects with eSigner EV code signing certificates. Hashes of the files are sent to SSL.com for signing so that the code itself is not sent. This is ideal where sensitive files need to be signed, but should not be sent over the wire for signing. CodeSignTool is also ideal for automated batch processes for high volume signings or integration into existing CI/CD pipeline workflows.

Docker image is used for code signing with CircleCI. Detailed information: https://github.com/sslcom/ci-images

# Usage (Job)

<!-- start usage -->
```yaml
- stage: sign
  # The job name
  name: sign-dotnet-artifacts
  # The Ubuntu distribution to use
  dist: bionic
  # Use docker command for signing   
  services:
    - docker
  # Defines environment variables for specific jobs.
  env:
    COMMAND="sign"
  # Use dotnet-cli to build the project
  language: csharp
  # Disable mono installation
  mono: none
  # Dotnet version to build the project
  dotnet: 3.1.419
  # Before script to run before building the project
  before_script: 
    # Created directories for artifacts
    - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
    - mkdir -p ${TRAVIS_BUILD_DIR}/packages
  # Script to build the project
  script: 
    # Docker Pull CodeSigner Docker Image
    - docker pull ghcr.io/sslcom/codesigner:latest
    # Sign artifact with CodeSigner docker image
    - docker run -i --rm --dns 8.8.8.8 --network host --volume ${TRAVIS_BUILD_DIR}/packages:/codesign/examples 
      --volume ${TRAVIS_BUILD_DIR}/artifacts:/codesign/output 
      -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} 
      -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} 
      -input_file_path=/codesign/examples/${PROJECT_NAME}.dll -output_dir_path=/codesign/output
  # Used to specify a list of files and directories that should be attached to the job if it succeeds.
  workspaces:
    use:
      - dotnet-artifacts
    create:
      name: dotnet-signed-artifacts
      paths:
        # Save signed artifact
        - ${TRAVIS_BUILD_DIR}/artifacts/${PROJECT_NAME}.dll
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
# The CPU Architecture to run the job on
arch: amd64

# Defines environment variables globally. Job level property overrides global variables
env:
  global:
    - PROJECT_NAME="HelloWorld"
    - PROJECT_VERSION="0.0.1"
    - DOTNET_VERSION="3.1"
    - TERM=dumb
    - ENVIRONMENT_NAME="TEST"

# Default language to run tests in
language: java

# The Ubuntu distribution to use
dist: bionic

# The Operating System to run the job on
os: linux

# Specifies the order of build stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

jobs:
  include:
    - stage: build
      # The job name
      name: build-dotnet
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Use dotnet-cli to build the project
      language: csharp
      # Disable mono installation
      mono: none
      # Dotnet version to build the project
      dotnet: 3.1.419
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
      # Script to build the project
      script: 
        # Build dotnet project with Release configuration
        - dotnet build dotnet/${PROJECT_NAME}.csproj -c Release
         # Copy built artifacts to artifacts directory
        - cp dotnet/bin/Release/netcoreapp${DOTNET_VERSION}/${PROJECT_NAME}-${PROJECT_VERSION}.dll ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.dll
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        create:
          name: dotnet-artifacts
          paths:
            # Save artifact in order to use signing job
            - ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.dll

    - stage: sign
      # The job name
      name: sign-dotnet-artifacts
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Defines environment variables for specific jobs.
      env:
        COMMAND="sign"
      # Use dotnet-cli to build the project
      language: csharp
      # Disable mono installation
      mono: none
      # Dotnet version to build the project
      dotnet: 3.1.419
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
      # Script to build the project
      script: 
        # Docker Pull CodeSigner Docker Image
        - docker pull ghcr.io/sslcom/codesigner:latest
        # Sign artifact with CodeSigner docker image
        - docker run -i --rm --dns 8.8.8.8 --network host --volume ${TRAVIS_BUILD_DIR}/packages:/codesign/examples 
          --volume ${TRAVIS_BUILD_DIR}/artifacts:/codesign/output 
          -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} 
          -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} 
          -input_file_path=/codesign/examples/${PROJECT_NAME}.dll -output_dir_path=/codesign/output
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        use:
          - dotnet-artifacts
        create:
          name: dotnet-signed-artifacts
          paths:
            # Save signed artifact
            - ${TRAVIS_BUILD_DIR}/artifacts/${PROJECT_NAME}.dll
```
<!-- end dotnet usage -->

## Java Code (Maven) JAR Signing Example Workflow

<!-- start maven usage -->
```yml
# The CPU Architecture to run the job on
arch: amd64

# Defines environment variables globally. Job level property overrides global variables
env:
  global:
    - PROJECT_NAME="HelloWorld"
    - PROJECT_VERSION="0.0.1"
    - TERM=dumb
    - ENVIRONMENT_NAME="TEST"

# Default language to run tests in
language: java

# The Ubuntu distribution to use
dist: bionic

# The Operating System to run the job on
os: linux

# Specifies the order of build stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

jobs:
  include:
        - stage: build
      # The job name
      name: build-maven-jar
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Use dotnet-cli to build the project
      language: java
      # Java version to build the project
      jdk: 
      - oraclejdk17
      # Cache m2 directory in order to speed up
      cache:
        directories:
        - $HOME/.m2
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
        # Set Maven build options
        - export MAVEN_CLI_OPTS="--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
      # Script to build the project
      script: 
        # Build Maven project with Maven Options
        - mvn $MAVEN_CLI_OPTS clean install -f java/pom.xml
         # Copy built artifacts to artifacts directory
        - cp java/target/${PROJECT_NAME}-${PROJECT_VERSION}.jar ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.jar
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        create:
          name: maven-jar-artifacts
          paths:
            # Save artifact in order to use signing job
            - ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.jar

    - stage: sign
      # The job name
      name: sign-maven-artifacts
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Defines environment variables for specific jobs.
      env:
        COMMAND="sign"
      # Use dotnet-cli to build the project
      language: java
      # Java version to build the project
      jdk: 
      - oraclejdk17
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
      # Script to build the project
      script: 
        # Docker Pull CodeSigner Docker Image
        - docker pull ghcr.io/sslcom/codesigner:latest
        # Sign artifact with CodeSigner docker image
        - docker run -i --rm --dns 8.8.8.8 --network host --volume ${TRAVIS_BUILD_DIR}/packages:/codesign/examples 
          --volume ${TRAVIS_BUILD_DIR}/artifacts:/codesign/output 
          -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} 
          -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} 
          -input_file_path=/codesign/examples/${PROJECT_NAME}.jar -output_dir_path=/codesign/output
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        use:
          - maven-jar-artifacts
        create:
          name: maven-signed-artifacts
          paths:
            # Save signed artifact
            - ${TRAVIS_BUILD_DIR}/artifacts/${PROJECT_NAME}.jar
```
<!-- end maven usage -->

## Java Code (Gradle) JAR Signing Example Workflow

<!-- start gradle usage -->
```yml
# The CPU Architecture to run the job on
arch: amd64

# Defines environment variables globally. Job level property overrides global variables
env:
  global:
    - PROJECT_NAME="HelloWorld"
    - PROJECT_VERSION="0.0.1"
    - TERM=dumb
    - ENVIRONMENT_NAME="TEST"

# Default language to run tests in
language: java

# The Ubuntu distribution to use
dist: bionic

# The Operating System to run the job on
os: linux

# Specifies the order of build stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

jobs:
  include:
    - stage: build
      # The job name
      name: build-gradle-jar
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Use dotnet-cli to build the project
      language: java
      # Java version to build the project
      jdk: 
      - oraclejdk11
      # Cache gradle directory in order to speed up
      cache:
        directories:
        - "$HOME/.gradle/caches/"
        - "$HOME/.gradle/wrapper/"
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
        # Set GRADLE options for building Java project
        - export GRADLE_OPTS="-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=4"
        - touch ${TRAVIS_BUILD_DIR}/java/gradle.properties
        - echo "${TRAVIS_BUILD_DIR}/java/gradle.properties" >> 'org.gradle.daemon=false'
        - echo "${TRAVIS_BUILD_DIR}/java/gradle.properties" >> 'org.gradle.workers.max=4'
      # Script to build the project
      script: 
        # Build Gradle project for Jar artifact
        - gradle clean build -p java -PsetupType=jar
         # Copy built artifacts to artifacts directory
        - cp java/build/libs/${PROJECT_NAME}-${PROJECT_VERSION}.jar ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.jar
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        create:
          name: gradle-jar-artifacts
          paths:
            # Save artifact in order to use signing job
            - ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.jar

    - stage: sign
      # The job name
      name: sign-gradle-artifacts
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Defines environment variables for specific jobs.
      env:
        COMMAND="sign"
      # Use dotnet-cli to build the project
      language: java
      # Java version to build the project
      jdk: 
      - oraclejdk17
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
      # Script to build the project
      script: 
        # Docker Pull CodeSigner Docker Image
        - docker pull ghcr.io/sslcom/codesigner:latest
        # Sign artifact with CodeSigner docker image
        - docker run -i --rm --dns 8.8.8.8 --network host --volume ${TRAVIS_BUILD_DIR}/packages:/codesign/examples 
          --volume ${TRAVIS_BUILD_DIR}/artifacts:/codesign/output 
          -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} 
          -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} 
          -input_file_path=/codesign/examples/${PROJECT_NAME}.jar -output_dir_path=/codesign/output
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        use:
          - gradle-jar-artifacts
        create:
          name: gradle-signed-artifacts
          paths:
            # Save signed artifact
            - ${TRAVIS_BUILD_DIR}/artifacts/${PROJECT_NAME}.jar
```
<!-- end gradle usage -->

## Java Code (Gradle) MSI Signing Example Workflow

<!-- start gradle usage -->
```yml
# The CPU Architecture to run the job on
arch: amd64

# Defines environment variables globally. Job level property overrides global variables
env:
  global:
    - PROJECT_NAME="HelloWorld"
    - PROJECT_VERSION="0.0.1"
    - TERM=dumb
    - ENVIRONMENT_NAME="TEST"

# Default language to run tests in
language: java

# The Ubuntu distribution to use
dist: bionic

# The Operating System to run the job on
os: linux

# Specifies the order of build stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

jobs:
  include:
    - stage: build
      # The job name
      name: build-gradle-msi
      # The operating system to run the job on
      os: windows
      # Use docker command for signing   
      services:
        - docker
      # Use default generic lang
      language: c
      # Cache gradle directory in order to speed up
      cache:
        directories:
        - "$HOME/.gradle/caches/"
        - "$HOME/.gradle/wrapper/"
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
        # Set GRADLE home for building Java project
        - export GRADLE_OPTS="-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=4"
        # Enable .NET Framework Core for building .NET project with WIX
        - powershell.exe -ExecutionPolicy Bypass Install-WindowsFeature Net-Framework-Core
        # Enable global confirmation for choco package installation
        - choco feature enable -n allowGlobalConfirmation
        # Install Dotnet3.5
        - choco install dotnet3.5 --version=3.5.20160716 -y
        # Install Oracle JDK 17.0.2
        - choco install oraclejdk --version=17.0.2 -y
        # Install Gradle 7.4
        - choco install gradle --version=7.4 -y
        # Install WIX 3.11.2
        - choco install wixtoolset --version=3.11.2 -y
        # Set JAVA HOME for building Java project
        - export JAVA_HOME="C:/PROGRA~1/Java/jdk-17.0.2"
        - export PATH="$JAVA_HOME/bin:$PATH"
        # Set Default JDK version for Gradle Build
        - powershell.exe -ExecutionPolicy Bypass New-Item ${TRAVIS_BUILD_DIR}/java/gradle.properties
        - powershell.exe -ExecutionPolicy Bypass Set-Content ${TRAVIS_BUILD_DIR}/java/gradle.properties "org.gradle.java.home=C:/PROGRA~1/Java/jdk-17.0.2"
        - powershell.exe -ExecutionPolicy Bypass Add-Content ${TRAVIS_BUILD_DIR}/java/gradle.properties "org.gradle.daemon=false"
        - powershell.exe -ExecutionPolicy Bypass Add-Content ${TRAVIS_BUILD_DIR}/java/gradle.properties "org.gradle.workers.max=4"
      # Script to build the project
      script: 
        # Build Gradle project for MSI artifact with jpackage
        - gradle build jpackage -x test --warning-mode all -p java -PsetupType=msi --no-daemon
        # Copy built artifacts to artifacts directory
        - powershell.exe -ExecutionPolicy Bypass Copy-Item "java/build/release/windows/*.msi" -Destination "${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.msi"
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        create:
          name: gradle-msi-artifacts
          paths:
            # Save artifact in order to use signing job
            - ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.msi

    - stage: sign
      # The job name
      name: sign-gradle-msi-artifacts
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Defines environment variables for specific jobs.
      env:
        COMMAND="sign"
      # Use dotnet-cli to build the project
      language: java
      # Java version to build the project
      jdk: 
      - oraclejdk17
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
        # Defines environment variables locally
        - export TRAVIS_REPO_OWNER=${TRAVIS_REPO_SLUG%/*}
        - export TRAVIS_REPO_NAME=${TRAVIS_REPO_SLUG#*/}
        # Copy artifact to artifacts directory (https://travis-ci.community/t/workspaces-do-not-work-nicely-with-cross-platform-builds/4461/3)
        - cp ${TRAVIS_BUILD_DIR}/C:/Users/travis/build/${TRAVIS_REPO_OWNER}/${TRAVIS_REPO_NAME}/packages/${PROJECT_NAME}.msi ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.msi
      # Script to build the project
      script: 
        # Docker Pull CodeSigner Docker Image
        - docker pull ghcr.io/sslcom/codesigner:latest
        # Sign artifact with CodeSigner docker image
        - docker run -i --rm --dns 8.8.8.8 --network host --volume ${TRAVIS_BUILD_DIR}/packages:/codesign/examples 
          --volume ${TRAVIS_BUILD_DIR}/artifacts:/codesign/output 
          -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} 
          -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} 
          -input_file_path=/codesign/examples/${PROJECT_NAME}.msi -output_dir_path=/codesign/output
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        use:
          - gradle-msi-artifacts
        create:
          name: gradle-signed-msi-artifacts
          paths:
            # Save signed artifact
            - ${TRAVIS_BUILD_DIR}/artifacts/${PROJECT_NAME}.msi
```
<!-- end gradle usage -->

## Java Code (Gradle) EXE Signing Example Workflow

<!-- start gradle usage -->
```yml
# The CPU Architecture to run the job on
arch: amd64

# Defines environment variables globally. Job level property overrides global variables
env:
  global:
    - PROJECT_NAME="HelloWorld"
    - PROJECT_VERSION="0.0.1"
    - TERM=dumb
    - ENVIRONMENT_NAME="TEST"

# Default language to run tests in
language: java

# The Ubuntu distribution to use
dist: bionic

# The Operating System to run the job on
os: linux

# Specifies the order of build stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

jobs:
  include:
    - stage: build
      # The job name
      name: build-gradle-exe
      # The operating system to run the job on
      os: windows
      # Use docker command for signing   
      services:
        - docker
      # Use default generic lang
      language: c
      # Cache gradle directory in order to speed up
      cache:
        directories:
        - "$HOME/.gradle/caches/"
        - "$HOME/.gradle/wrapper/"
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
        # Set GRADLE home for building Java project
        - export GRADLE_OPTS="-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=4"
        # Enable .NET Framework Core for building .NET project with WIX
        - powershell.exe -ExecutionPolicy Bypass Install-WindowsFeature Net-Framework-Core
        # Enable global confirmation for choco package installation
        - choco feature enable -n allowGlobalConfirmation
        # Install Dotnet3.5
        - choco install dotnet3.5 --version=3.5.20160716 -y
        # Install Oracle JDK 17.0.2
        - choco install oraclejdk --version=17.0.2 -y
        # Install Gradle 7.4
        - choco install gradle --version=7.4 -y
        # Install WIX 3.11.2
        - choco install wixtoolset --version=3.11.2 -y
        # Set JAVA HOME for building Java project
        - export JAVA_HOME="C:/PROGRA~1/Java/jdk-17.0.2"
        - export PATH="$JAVA_HOME/bin:$PATH"
        # Set Default JDK version for Gradle Build
        - powershell.exe -ExecutionPolicy Bypass New-Item ${TRAVIS_BUILD_DIR}/java/gradle.properties
        - powershell.exe -ExecutionPolicy Bypass Set-Content ${TRAVIS_BUILD_DIR}/java/gradle.properties "org.gradle.java.home=C:/PROGRA~1/Java/jdk-17.0.2"
        - powershell.exe -ExecutionPolicy Bypass Add-Content ${TRAVIS_BUILD_DIR}/java/gradle.properties "org.gradle.daemon=false"
        - powershell.exe -ExecutionPolicy Bypass Add-Content ${TRAVIS_BUILD_DIR}/java/gradle.properties "org.gradle.workers.max=4"
      # Script to build the project
      script: 
        # Build Gradle project for EXE artifact with jpackage
        - gradle build jpackage -x test --warning-mode all -p java -PsetupType=exe --no-daemon
        # Copy built artifacts to artifacts directory
        - powershell.exe -ExecutionPolicy Bypass Copy-Item "java/build/release/windows/*.exe" -Destination "${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.exe"
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        create:
          name: gradle-exe-artifacts
          paths:
            # Save artifact in order to use signing job
            - ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.exe

    - stage: sign
      # The job name
      name: sign-gradle-exe-artifacts
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Defines environment variables for specific jobs.
      env:
        COMMAND="sign"
      # Use dotnet-cli to build the project
      language: java
      # Java version to build the project
      jdk: 
      - oraclejdk17
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
        # Defines environment variables locally
        - export TRAVIS_REPO_OWNER=${TRAVIS_REPO_SLUG%/*}
        - export TRAVIS_REPO_NAME=${TRAVIS_REPO_SLUG#*/}
        # Copy artifact to artifacts directory (https://travis-ci.community/t/workspaces-do-not-work-nicely-with-cross-platform-builds/4461/3)
        - cp ${TRAVIS_BUILD_DIR}/C:/Users/travis/build/${TRAVIS_REPO_OWNER}/${TRAVIS_REPO_NAME}/packages/${PROJECT_NAME}.exe ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.exe
      # Script to build the project
      script: 
        # Docker Pull CodeSigner Docker Image
        - docker pull ghcr.io/sslcom/codesigner:latest
        # Sign artifact with CodeSigner docker image
        - docker run -i --rm --dns 8.8.8.8 --network host --volume ${TRAVIS_BUILD_DIR}/packages:/codesign/examples 
          --volume ${TRAVIS_BUILD_DIR}/artifacts:/codesign/output 
          -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} 
          -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} 
          -input_file_path=/codesign/examples/${PROJECT_NAME}.exe -output_dir_path=/codesign/output
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        use:
          - gradle-exe-artifacts
        create:
          name: gradle-signed-exe-artifacts
          paths:
            # Save signed artifact
            - ${TRAVIS_BUILD_DIR}/artifacts/${PROJECT_NAME}.exe
```
<!-- end gradle usage -->

## Powershell PS1 Signing Example Workflow

<!-- start powershell usage -->
```yml
# The CPU Architecture to run the job on
arch: amd64

# Defines environment variables globally. Job level property overrides global variables
env:
  global:
    - PROJECT_NAME="HelloWorld"
    - PROJECT_VERSION="0.0.1"
    - TERM=dumb
    - ENVIRONMENT_NAME="TEST"

# Default language to run tests in
language: java

# The Ubuntu distribution to use
dist: bionic

# The Operating System to run the job on
os: linux

# Specifies the order of build stages. All jobs in one stage must complete before next stage is executed.
stages:
  - build
  - sign

jobs:
  include:
    - stage: sign
      # The job name
      name: sign-ps1-artifacts
      # The Ubuntu distribution to use
      dist: bionic
      # Use docker command for signing   
      services:
        - docker
      # Defines environment variables for specific jobs.
      env:
        COMMAND="sign"
      # Use dotnet-cli to build the project
      language: java
      # Java version to build the project
      jdk: 
      - oraclejdk17
      # Before script to run before building the project
      before_script: 
        # Created directories for artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/artifacts
        - mkdir -p ${TRAVIS_BUILD_DIR}/packages
        # Copy ps1 script for signing
        - cp powershell/${PROJECT_NAME}.ps1 ${TRAVIS_BUILD_DIR}/packages/${PROJECT_NAME}.ps1
      # Script to build the project
      script: 
        # Docker Pull CodeSigner Docker Image
        - docker pull ghcr.io/sslcom/codesigner:latest
        # Sign artifact with CodeSigner docker image
        - docker run -i --rm --dns 8.8.8.8 --network host --volume ${TRAVIS_BUILD_DIR}/packages:/codesign/examples 
          --volume ${TRAVIS_BUILD_DIR}/artifacts:/codesign/output 
          -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} 
          -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/sslcom/codesigner:latest ${COMMAND} 
          -input_file_path=/codesign/examples/${PROJECT_NAME}.ps1 -output_dir_path=/codesign/output
      # Used to specify a list of files and directories that should be attached to the job if it succeeds.
      workspaces:
        use:
          - ps1-artifacts
        create:
          name: signed-ps1-artifacts
          paths:
            # Save signed artifact
            - ${TRAVIS_BUILD_DIR}/artifacts/${PROJECT_NAME}.ps1
```
<!-- end powershell usage -->

# CodeSignTool Guide

* https://www.ssl.com/guide/esigner-codesigntool-command-guide
* https://www.ssl.com/guide/remote-ev-code-signing-with-esigner/
