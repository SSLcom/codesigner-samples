# Sign with CodeSignTool

[![GitHub Actions Docker Status](https://github.com/bayrakmustafa/codesigner-docker/workflows/Docker%20Image%20CI/badge.svg)](https://github.com/bayrakmustafa/codesigner-docker)
[![CircleCI](https://circleci.com/gh/bayrakmustafa/codesigner-samples/tree/main.svg?style=shield)](https://circleci.com/gh/bayrakmustafa/codesigner-samples/tree/main)

CodeSignTool is a secure, privacy-oriented multi-platform Java command line utility for remotely signing Microsoft Authenticode and Java code objects with eSigner EV code signing certificates. Hashes of the files are sent to SSL.com for signing so that the code itself is not sent. This is ideal where sensitive files need to be signed, but should not be sent over the wire for signing. CodeSignTool is also ideal for automated batch processes for high volume signings or integration into existing CI/CD pipeline workflows.

Docker image is used for code signing with CircleCI. Detailed information: https://github.com/bayrakmustafa/codesigner-docker

# Usage (Job)

<!-- start usage -->
```yaml
codesigner-sign-artifact:
    # Create an environment variable
    environment:
        REGISTRY_USERNAME: bayrakmustafa
        ENVIRONMENT_NAME: TEST # TEST, PROD
        COMMAND: sign
        WORKSPACE: /home/circleci/project

    # Artifact name for signing
    parameters:
        artifact-name:
          type: string
          default: ''

    # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
    # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
    # Be sure to update the Docker image tag below to openjdk version of your application.
    # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
    docker:
      - image: cimg/openjdk:17.0.3

    # Working directory for the job on Circle-CI
    working_directory: /home/circleci/project

    # Add steps to the job
    # See: https://circleci.com/docs/2.0/configuration-reference/#steps
    steps:
      # 1) Create Artifact Directory to store signed and unsigned artifact files
      - run:
          name: Create Artifacts Directory
          command: |
            mkdir -p ${WORKSPACE}/artifacts
            mkdir -p ${WORKSPACE}/packages

      # 2) Attach to Workspace in order to access the artifact file
      - attach_workspace:
          at: /home/circleci/project

      # 3) Enable Docker for CodeSigner on Circle-CI
      - setup_remote_docker:
          name: Setup Remote Docker
          version: 19.03.13
          docker_layer_caching: true

      # 4) Pull Codesigner Docker Image From Github Registry
      - run:
          name: Docker Pull Image
          command: |
            docker pull ghcr.io/bayrakmustafa/codesigner:latest
            docker pull alpine:3.4

      # 5) This is the step where the created DLL, JAR, EXE, MSI, PS1 (artifact) files will be signed with CodeSignTool.
      - run:
          name: Sign Artifact File
          command: |
            docker create -v /codesign/packages  --name codesign-in  alpine:3.4 /bin/true
            docker create -v /codesign/artifacts --name codesign-out alpine:3.4 /bin/true
            docker cp ${WORKSPACE}/packages/<< parameters.artifact-name >> codesign-in:/codesign/packages
            docker run -i --rm --dns 8.8.8.8 --network host --volumes-from codesign-in --volumes-from codesign-out 
              -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} 
              -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} 
              ghcr.io/bayrakmustafa/codesigner:latest ${COMMAND} 
              -input_file_path=/codesign/packages/<< parameters.artifact-name >> -output_dir_path=/codesign/artifacts
            docker cp codesign-out:/codesign/artifacts/<< parameters.artifact-name >> ${WORKSPACE}/artifacts/<< parameters.artifact-name >>

      # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
      - store_artifacts:
          name: Upload Signed Files
          path: /home/circleci/project/artifacts/<< parameters.artifact-name >>
          destination: << parameters.artifact-name >>
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
# Set the CI version.
version: 2.1

# Invoke jobs via workflows
# See: https://circleci.com/docs/2.0/configuration-reference/#workflows
# Workflows orchestrate a set of jobs to be run. The jobs for this pipeline are # configured below
workflows:
  # The name of the workflow.
  codesigner-dotnet-dll:
    # Inside the workflow, you define the jobs you want to run.
    jobs:
      - dotnet-build-dll
      - codesigner-sign-artifact:
          requires:
            - dotnet-build-dll
          artifact-name: 'HelloWorld.dll'

# Define a job to be invoked later in a workflow.
# See: https://circleci.com/docs/2.0/configuration-reference/#jobs
jobs:
  # Below is the definition of your job to build and sign dll artifact
  dotnet-build-dll:
    # Create an environment variable
    environment:
        PROJECT_NAME: HelloWorld
        PROJECT_VERSION: 0.0.1
        DOTNET_VERSION: 3.1
        WORKSPACE: /home/circleci/project

    # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
    # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
    docker:
      - image: mcr.microsoft.com/dotnet/sdk:3.1-bullseye

    # Working directory for the job on Circle-CI
    working_directory: /home/circleci/project
    
    # Add steps to the job
    # See: https://circleci.com/docs/2.0/configuration-reference/#steps
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - checkout

      # 2) Create Artifact Directory to store signed and unsigned artifact files
      - run:
          name: Create Artifacts Directory
          command: |
            mkdir -p ${WORKSPACE}/artifacts
            mkdir -p ${WORKSPACE}/packages

      # 3) Build a dotnet project or solution and all of its dependencies.
      - run:
          name: Compile Dotnet Library with Dotnet Core
          command: | 
            dotnet build dotnet/${PROJECT_NAME}.csproj -c Release

      # 4) Copy artifact to artifacts directory
      - run:
          name: Copy Artifact File
          command: | 
            cp dotnet/bin/Release/netcoreapp${DOTNET_VERSION}/${PROJECT_NAME}-${PROJECT_VERSION}.dll ${WORKSPACE}/packages/HelloWorld.dll

      # 5) Persist the artifact to the artifacts directory for signing
      - persist_to_workspace:
          root: .
          paths:
            - packages/*
  
  # Below is the definition of your job to sign artifacts
  codesigner-sign-artifact:
      # Create an environment variable
      environment:
          REGISTRY_USERNAME: bayrakmustafa
          ENVIRONMENT_NAME: TEST
          COMMAND: sign
          WORKSPACE: /home/circleci/project

      # Artifact name for signing
      parameters:
          artifact-name:
            type: string
            default: ''

      # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
      # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
      # Be sure to update the Docker image tag below to openjdk version of your application.
      # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
      docker:
        - image: cimg/openjdk:17.0.3

      # Working directory for the job on Circle-CI
      working_directory: /home/circleci/project

      # Add steps to the job
      # See: https://circleci.com/docs/2.0/configuration-reference/#steps
      steps:
        # 1) Create Artifact Directory to store signed and unsigned artifact files
        - run:
            name: Create Artifacts Directory
            command: |
              mkdir -p ${WORKSPACE}/artifacts
              mkdir -p ${WORKSPACE}/packages

        # 2) Attach to Workspace in order to access the artifact file
        - attach_workspace:
            at: /home/circleci/project

        # 3) Enable Docker for CodeSigner on Circle-CI
        - setup_remote_docker:
            name: Setup Remote Docker
            version: 19.03.13
            docker_layer_caching: true

        # 4) Pull Codesigner Docker Image From Github Registry
        - run:
            name: Docker Pull Image
            command: |
              docker pull ghcr.io/bayrakmustafa/codesigner:latest
              docker pull alpine:3.4

        # 5) This is the step where the created DLL, JAR, EXE, MSI, PS1 (artifact) files will be signed with CodeSignTool.
        - run:
            name: Sign Artifact File
            command: |
              docker create -v /codesign/packages  --name codesign-in  alpine:3.4 /bin/true
              docker create -v /codesign/artifacts --name codesign-out alpine:3.4 /bin/true
              docker cp ${WORKSPACE}/packages/<< parameters.artifact-name >> codesign-in:/codesign/packages
              docker run -i --rm --dns 8.8.8.8 --network host --volumes-from codesign-in --volumes-from codesign-out -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/bayrakmustafa/codesigner:latest ${COMMAND} -input_file_path=/codesign/packages/<< parameters.artifact-name >> -output_dir_path=/codesign/artifacts
              docker cp codesign-out:/codesign/artifacts/<< parameters.artifact-name >> ${WORKSPACE}/artifacts/<< parameters.artifact-name >>

        # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
        - store_artifacts:
            name: Upload Signed Files
            path: /home/circleci/project/artifacts/<< parameters.artifact-name >>
            destination: << parameters.artifact-name >>
```
<!-- end dotnet usage -->

## Java Code (Maven) JAR Signing Example Workflow

<!-- start maven usage -->
```yml
# Set the CI version.
version: 2.1

# Invoke jobs via workflows
# See: https://circleci.com/docs/2.0/configuration-reference/#workflows
# Workflows orchestrate a set of jobs to be run. The jobs for this pipeline are # configured below
workflows:
  # The name of the workflow.
  codesigner-maven-jar:
    # Inside the workflow, you define the jobs you want to run.
    jobs:
      - maven-build-jar
      - codesigner-sign-artifact:
          requires:
            - maven-build-jar
          artifact-name: 'HelloWorld-Maven.jar'

# Define a job to be invoked later in a workflow.
# See: https://circleci.com/docs/2.0/configuration-reference/#jobs
jobs:
  # Below is the definition of your job to build and sign jar artifact with Maven
  maven-build-jar:
    # Create an environment variable
    environment:
        PROJECT_NAME: HelloWorld
        PROJECT_VERSION: 0.0.1
        WORKSPACE: /home/circleci/project

    # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
    # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
    # Be sure to update the Docker image tag below to openjdk version of your application.
    # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
    docker:
      - image: cimg/openjdk:17.0.3

    # Working directory for the job on Circle-CI
    working_directory: /home/circleci/project
    
    # Add steps to the job
    # See: https://circleci.com/docs/2.0/configuration-reference/#steps
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - checkout

      # 2) Create Artifact Directory to store signed and unsigned artifact files
      - run:
          name: Create Artifacts Directory
          command: |
            mkdir -p ${WORKSPACE}/artifacts
            mkdir -p ${WORKSPACE}/packages

      # 3) Build a maven project or solution and all of its dependencies.
      - run:
          name: Compile Java Library with Maven
          command: | 
            mvn clean install -f java/pom.xml

      # 4) Copy artifact to artifacts directory
      - run:
          name: Copy Artifact File
          command: | 
            cp java/target/${PROJECT_NAME}-${PROJECT_VERSION}.jar ${WORKSPACE}/packages/HelloWorld-Maven.jar

      # 5) Persist the artifact to the artifacts directory for signing
      - persist_to_workspace:
          root: .
          paths:
            - packages/*
  
  # Below is the definition of your job to sign artifacts
  codesigner-sign-artifact:
      # Create an environment variable
      environment:
          REGISTRY_USERNAME: bayrakmustafa
          ENVIRONMENT_NAME: TEST
          COMMAND: sign
          WORKSPACE: /home/circleci/project

      # Artifact name for signing
      parameters:
          artifact-name:
            type: string
            default: ''

      # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
      # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
      # Be sure to update the Docker image tag below to openjdk version of your application.
      # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
      docker:
        - image: cimg/openjdk:17.0.3

      # Working directory for the job on Circle-CI
      working_directory: /home/circleci/project

      # Add steps to the job
      # See: https://circleci.com/docs/2.0/configuration-reference/#steps
      steps:
        # 1) Create Artifact Directory to store signed and unsigned artifact files
        - run:
            name: Create Artifacts Directory
            command: |
              mkdir -p ${WORKSPACE}/artifacts
              mkdir -p ${WORKSPACE}/packages

        # 2) Attach to Workspace in order to access the artifact file
        - attach_workspace:
            at: /home/circleci/project

        # 3) Enable Docker for CodeSigner on Circle-CI
        - setup_remote_docker:
            name: Setup Remote Docker
            version: 19.03.13
            docker_layer_caching: true

        # 4) Pull Codesigner Docker Image From Github Registry
        - run:
            name: Docker Pull Image
            command: |
              docker pull ghcr.io/bayrakmustafa/codesigner:latest
              docker pull alpine:3.4

        # 5) This is the step where the created DLL, JAR, EXE, MSI, PS1 (artifact) files will be signed with CodeSignTool.
        - run:
            name: Sign Artifact File
            command: |
              docker create -v /codesign/packages  --name codesign-in  alpine:3.4 /bin/true
              docker create -v /codesign/artifacts --name codesign-out alpine:3.4 /bin/true
              docker cp ${WORKSPACE}/packages/<< parameters.artifact-name >> codesign-in:/codesign/packages
              docker run -i --rm --dns 8.8.8.8 --network host --volumes-from codesign-in --volumes-from codesign-out -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/bayrakmustafa/codesigner:latest ${COMMAND} -input_file_path=/codesign/packages/<< parameters.artifact-name >> -output_dir_path=/codesign/artifacts
              docker cp codesign-out:/codesign/artifacts/<< parameters.artifact-name >> ${WORKSPACE}/artifacts/<< parameters.artifact-name >>

        # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
        - store_artifacts:
            name: Upload Signed Files
            path: /home/circleci/project/artifacts/<< parameters.artifact-name >>
            destination: << parameters.artifact-name >>
```
<!-- end maven usage -->

## Java Code (Gradle) JAR Signing Example Workflow

<!-- start gradle usage -->
```yml
# Set the CI version.
version: 2.1

# Invoke jobs via workflows
# See: https://circleci.com/docs/2.0/configuration-reference/#workflows
# Workflows orchestrate a set of jobs to be run. The jobs for this pipeline are # configured below
workflows:
  # The name of the workflow.
  codesigner-gradle-jar:
    # Inside the workflow, you define the jobs you want to run.
    jobs:
      - gradle-build-jar
      - codesigner-sign-artifact:
          requires:
            - gradle-build-jar
          artifact-name: 'HelloWorld-Gradle.jar'

# Define a job to be invoked later in a workflow.
# See: https://circleci.com/docs/2.0/configuration-reference/#jobs
jobs:
  # Below is the definition of your job to build and sign jar artifact with Gradle
  gradle-build-jar:
    # Specify the execution environment. You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
    # See: https://circleci.com/docs/2.0/configuration-reference/#docker-machine-macos-windows-executor
    docker:
      # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
      # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
      # Be sure to update the Docker image tag below to openjdk version of your application.
      # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
      - image: cimg/openjdk:17.0.3

    # Create an environment variable
    environment:
      # Customize the JVM maximum heap limit
      JVM_OPTS: -Xmx3200m
      TERM: dumb
      GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=4"
      PROJECT_NAME: HelloWorld
      PROJECT_VERSION: 0.0.1
      WORKSPACE: /home/circleci/project

    # Working directory for the job on Circle-CI
    working_directory: /home/circleci/project

    # Add steps to the job
    # See: https://circleci.com/docs/2.0/configuration-reference/#steps
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - checkout

      # 2) Create Artifact Directory to store signed and unsigned artifact files
      - run:
          name: Create Artifacts Directory
          command: |
            mkdir -p ${WORKSPACE}/artifacts
            mkdir -p ${WORKSPACE}/packages

      # 3) Build a gradle project or solution and all of its dependencies.
      - run:
          name: Compile Java Library with Gradle
          command: |
            gradle clean build -p java -PsetupType=jar

      # 4) Copy artifact to artifacts directory
      - run:
          name: Copy Artifact File
          command: | 
            cp java/build/libs/${PROJECT_NAME}-${PROJECT_VERSION}.jar ${WORKSPACE}/packages/HelloWorld-Gradle.jar

      # 5) Persist the artifact to the artifacts directory for signing
      - persist_to_workspace:
          root: .
          paths:
            - packages/*
  
  # Below is the definition of your job to sign artifacts
  codesigner-sign-artifact:
      # Create an environment variable
      environment:
          REGISTRY_USERNAME: bayrakmustafa
          ENVIRONMENT_NAME: TEST
          COMMAND: sign
          WORKSPACE: /home/circleci/project

      # Artifact name for signing
      parameters:
          artifact-name:
            type: string
            default: ''

      # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
      # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
      # Be sure to update the Docker image tag below to openjdk version of your application.
      # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
      docker:
        - image: cimg/openjdk:17.0.3

      # Working directory for the job on Circle-CI
      working_directory: /home/circleci/project

      # Add steps to the job
      # See: https://circleci.com/docs/2.0/configuration-reference/#steps
      steps:
        # 1) Create Artifact Directory to store signed and unsigned artifact files
        - run:
            name: Create Artifacts Directory
            command: |
              mkdir -p ${WORKSPACE}/artifacts
              mkdir -p ${WORKSPACE}/packages

        # 2) Attach to Workspace in order to access the artifact file
        - attach_workspace:
            at: /home/circleci/project

        # 3) Enable Docker for CodeSigner on Circle-CI
        - setup_remote_docker:
            name: Setup Remote Docker
            version: 19.03.13
            docker_layer_caching: true

        # 4) Pull Codesigner Docker Image From Github Registry
        - run:
            name: Docker Pull Image
            command: |
              docker pull ghcr.io/bayrakmustafa/codesigner:latest
              docker pull alpine:3.4

        # 5) This is the step where the created DLL, JAR, EXE, MSI, PS1 (artifact) files will be signed with CodeSignTool.
        - run:
            name: Sign Artifact File
            command: |
              docker create -v /codesign/packages  --name codesign-in  alpine:3.4 /bin/true
              docker create -v /codesign/artifacts --name codesign-out alpine:3.4 /bin/true
              docker cp ${WORKSPACE}/packages/<< parameters.artifact-name >> codesign-in:/codesign/packages
              docker run -i --rm --dns 8.8.8.8 --network host --volumes-from codesign-in --volumes-from codesign-out -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/bayrakmustafa/codesigner:latest ${COMMAND} -input_file_path=/codesign/packages/<< parameters.artifact-name >> -output_dir_path=/codesign/artifacts
              docker cp codesign-out:/codesign/artifacts/<< parameters.artifact-name >> ${WORKSPACE}/artifacts/<< parameters.artifact-name >>

        # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
        - store_artifacts:
            name: Upload Signed Files
            path: /home/circleci/project/artifacts/<< parameters.artifact-name >>
            destination: << parameters.artifact-name >>
```
<!-- end gradle usage -->

## Java Code (Gradle) MSI Signing Example Workflow

<!-- start gradle usage -->
```yml
# Set the CI version.
version: 2.1

# Windows Orbs
orbs:
  win: circleci/windows@4.1.1

# Invoke jobs via workflows
# See: https://circleci.com/docs/2.0/configuration-reference/#workflows
# Workflows orchestrate a set of jobs to be run. The jobs for this pipeline are # configured below
workflows:
  # The name of the workflow.
  codesigner-gradle-msi:
    # Inside the workflow, you define the jobs you want to run.
    jobs:
      - gradle-build-msi
      - codesigner-sign-artifact:
          requires:
            - gradle-build-msi
          artifact-name: 'HelloWorld.msi'

# Define a job to be invoked later in a workflow.
# See: https://circleci.com/docs/2.0/configuration-reference/#jobs
jobs:
  # Below is the definition of your job to build and sign msi artifact with Gradle
  gradle-build-msi:
    # Specify the execution environment. You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
    # See: https://circleci.com/docs/2.0/configuration-reference/#docker-machine-macos-windows-executor
    executor: 
      name: win/default
      size: large
      variant: vs2019

    # Create an environment variable
    environment:
      # Customize the JVM maximum heap limit
      JVM_OPTS: -Xmx3200m
      TERM: dumb
      GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=4"
      PROJECT_NAME: HelloWorld
      PROJECT_VERSION: 0.0.1
      WORKSPACE: /home/circleci/project

    # Working directory for the job on Circle-CI
    working_directory: /home/circleci/project

    # Add steps to the job
    # See: https://circleci.com/docs/2.0/configuration-reference/#steps
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - checkout

      # 2) Install Java, Gradle And WixtoolSet for Build
      - run:
          name: Install Java, Gradle And WixtoolSet
          command: |
            choco install oraclejdk --version=17.0.2
            choco install gradle --version=7.4
            choco install wixtoolset --version=3.11.2

      # 3) Create Artifact Directory to store signed and unsigned artifact files
      - run:
          name: Create Artifacts Directory
          command: |
            mkdir -p /home/circleci/project\artifacts
            mkdir -p /home/circleci/project\packages

      # 4) Prepare the Gradle properties for JDK
      - run:
          name: Prepare Gradle Properties
          command: |
            New-Item /home/circleci/project/java/gradle.properties
            Set-Content /home/circleci/project/java/gradle.properties 'org.gradle.java.home=C:/Program Files/Java/jdk-17.0.2'

      # 5) Build a gradle project or solution and all of its dependencies.
      - run:
          name: Compile Java Library with Gradle
          command: |
            gradle build jpackage -x test --warning-mode all -p java -PsetupType=msi

      # 6) Copy artifact to artifacts directory
      - run:
          name: Copy Artifact File
          command: | 
            Copy-Item "java/build/release/windows/*.msi" -Destination "/home/circleci/project/packages/HelloWorld.msi"

      # 7) Persist the artifact to the artifacts directory for signing
      - persist_to_workspace:
          root: .
          paths:
            - packages/*
  
  # Below is the definition of your job to sign artifacts
  codesigner-sign-artifact:
      # Create an environment variable
      environment:
          REGISTRY_USERNAME: bayrakmustafa
          ENVIRONMENT_NAME: TEST
          COMMAND: sign
          WORKSPACE: /home/circleci/project

      # Artifact name for signing
      parameters:
          artifact-name:
            type: string
            default: ''

      # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
      # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
      # Be sure to update the Docker image tag below to openjdk version of your application.
      # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
      docker:
        - image: cimg/openjdk:17.0.3

      # Working directory for the job on Circle-CI
      working_directory: /home/circleci/project

      # Add steps to the job
      # See: https://circleci.com/docs/2.0/configuration-reference/#steps
      steps:
        # 1) Create Artifact Directory to store signed and unsigned artifact files
        - run:
            name: Create Artifacts Directory
            command: |
              mkdir -p ${WORKSPACE}/artifacts
              mkdir -p ${WORKSPACE}/packages

        # 2) Attach to Workspace in order to access the artifact file
        - attach_workspace:
            at: /home/circleci/project

        # 3) Enable Docker for CodeSigner on Circle-CI
        - setup_remote_docker:
            name: Setup Remote Docker
            version: 19.03.13
            docker_layer_caching: true

        # 4) Pull Codesigner Docker Image From Github Registry
        - run:
            name: Docker Pull Image
            command: |
              docker pull ghcr.io/bayrakmustafa/codesigner:latest
              docker pull alpine:3.4

        # 5) This is the step where the created DLL, JAR, EXE, MSI, PS1 (artifact) files will be signed with CodeSignTool.
        - run:
            name: Sign Artifact File
            command: |
              docker create -v /codesign/packages  --name codesign-in  alpine:3.4 /bin/true
              docker create -v /codesign/artifacts --name codesign-out alpine:3.4 /bin/true
              docker cp ${WORKSPACE}/packages/<< parameters.artifact-name >> codesign-in:/codesign/packages
              docker run -i --rm --dns 8.8.8.8 --network host --volumes-from codesign-in --volumes-from codesign-out -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/bayrakmustafa/codesigner:latest ${COMMAND} -input_file_path=/codesign/packages/<< parameters.artifact-name >> -output_dir_path=/codesign/artifacts
              docker cp codesign-out:/codesign/artifacts/<< parameters.artifact-name >> ${WORKSPACE}/artifacts/<< parameters.artifact-name >>

        # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
        - store_artifacts:
            name: Upload Signed Files
            path: /home/circleci/project/artifacts/<< parameters.artifact-name >>
            destination: << parameters.artifact-name >>
```
<!-- end gradle usage -->

## Java Code (Gradle) EXE Signing Example Workflow

<!-- start gradle usage -->
```yml
# Set the CI version.
version: 2.1

# Windows Orbs
orbs:
  win: circleci/windows@4.1.1

# Invoke jobs via workflows
# See: https://circleci.com/docs/2.0/configuration-reference/#workflows
# Workflows orchestrate a set of jobs to be run. The jobs for this pipeline are # configured below
workflows:
  # The name of the workflow.
  codesigner-gradle-exe:
    # Inside the workflow, you define the jobs you want to run.
    jobs:
      - gradle-build-exe
      - codesigner-sign-artifact:
          requires:
            - gradle-build-exe
          artifact-name: 'HelloWorld.exe'

# Define a job to be invoked later in a workflow.
# See: https://circleci.com/docs/2.0/configuration-reference/#jobs
jobs:
  # Below is the definition of your job to build and sign exe artifact with Gradle
  gradle-build-exe:
    # Specify the execution environment. You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
    # See: https://circleci.com/docs/2.0/configuration-reference/#docker-machine-macos-windows-executor
    executor: 
      name: win/default
      size: large
      variant: vs2019

    # Create an environment variable
    environment:
      # Customize the JVM maximum heap limit
      JVM_OPTS: -Xmx3200m
      TERM: dumb
      GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.workers.max=4"
      PROJECT_NAME: HelloWorld
      PROJECT_VERSION: 0.0.1
      WORKSPACE: /home/circleci/project

    # Working directory for the job on Circle-CI
    working_directory: /home/circleci/project

    # Add steps to the job
    # See: https://circleci.com/docs/2.0/configuration-reference/#steps
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - checkout

      # 2) Install Java, Gradle And WixtoolSet for Build
      - run:
          name: Install Java, Gradle And WixtoolSet
          command: |
            choco install oraclejdk --version=17.0.2
            choco install gradle --version=7.4
            choco install wixtoolset --version=3.11.2

      # 3) Create Artifact Directory to store signed and unsigned artifact files
      - run:
          name: Create Artifacts Directory
          command: |
            mkdir -p /home/circleci/project\artifacts
            mkdir -p /home/circleci/project\packages

      # 4) Prepare the Gradle properties for JDK
      - run:
          name: Prepare Gradle Properties
          command: |
            New-Item /home/circleci/project/java/gradle.properties
            Set-Content /home/circleci/project/java/gradle.properties 'org.gradle.java.home=C:/Program Files/Java/jdk-17.0.2'

      # 5) Build a gradle project or solution and all of its dependencies.
      - run:
          name: Compile Java Library with Gradle
          command: |
            gradle build jpackage -x test --warning-mode all -p java -PsetupType=exe

      # 6) Copy artifact to artifacts directory
      - run:
          name: Copy Artifact File
          command: | 
            Copy-Item "java/build/release/windows/*.exe" -Destination "/home/circleci/project/packages/HelloWorld.exe"

      # 7) Persist the artifact to the artifacts directory for signing
      - persist_to_workspace:
          root: .
          paths:
            - packages/*
  
  # Below is the definition of your job to sign artifacts
  codesigner-sign-artifact:
      # Create an environment variable
      environment:
          REGISTRY_USERNAME: bayrakmustafa
          ENVIRONMENT_NAME: TEST
          COMMAND: sign
          WORKSPACE: /home/circleci/project

      # Artifact name for signing
      parameters:
          artifact-name:
            type: string
            default: ''

      # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
      # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
      # Be sure to update the Docker image tag below to openjdk version of your application.
      # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
      docker:
        - image: cimg/openjdk:17.0.3

      # Working directory for the job on Circle-CI
      working_directory: /home/circleci/project

      # Add steps to the job
      # See: https://circleci.com/docs/2.0/configuration-reference/#steps
      steps:
        # 1) Create Artifact Directory to store signed and unsigned artifact files
        - run:
            name: Create Artifacts Directory
            command: |
              mkdir -p ${WORKSPACE}/artifacts
              mkdir -p ${WORKSPACE}/packages

        # 2) Attach to Workspace in order to access the artifact file
        - attach_workspace:
            at: /home/circleci/project

        # 3) Enable Docker for CodeSigner on Circle-CI
        - setup_remote_docker:
            name: Setup Remote Docker
            version: 19.03.13
            docker_layer_caching: true

        # 4) Pull Codesigner Docker Image From Github Registry
        - run:
            name: Docker Pull Image
            command: |
              docker pull ghcr.io/bayrakmustafa/codesigner:latest
              docker pull alpine:3.4

        # 5) This is the step where the created DLL, JAR, EXE, MSI, PS1 (artifact) files will be signed with CodeSignTool.
        - run:
            name: Sign Artifact File
            command: |
              docker create -v /codesign/packages  --name codesign-in  alpine:3.4 /bin/true
              docker create -v /codesign/artifacts --name codesign-out alpine:3.4 /bin/true
              docker cp ${WORKSPACE}/packages/<< parameters.artifact-name >> codesign-in:/codesign/packages
              docker run -i --rm --dns 8.8.8.8 --network host --volumes-from codesign-in --volumes-from codesign-out -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/bayrakmustafa/codesigner:latest ${COMMAND} -input_file_path=/codesign/packages/<< parameters.artifact-name >> -output_dir_path=/codesign/artifacts
              docker cp codesign-out:/codesign/artifacts/<< parameters.artifact-name >> ${WORKSPACE}/artifacts/<< parameters.artifact-name >>

        # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
        - store_artifacts:
            name: Upload Signed Files
            path: /home/circleci/project/artifacts/<< parameters.artifact-name >>
            destination: << parameters.artifact-name >>
```
<!-- end gradle usage -->

## Powershell PS1 Signing Example Workflow

<!-- start powershell usage -->
```yml
# Set the CI version.
version: 2.1

# Invoke jobs via workflows
# See: https://circleci.com/docs/2.0/configuration-reference/#workflows
# Workflows orchestrate a set of jobs to be run. The jobs for this pipeline are # configured below
workflows:
  # The name of the workflow.
  codesigner-powershell-ps1:
    # Inside the workflow, you define the jobs you want to run.
    jobs:
      - powershell-build-ps1
      - codesigner-sign-artifact:
          requires:
            - powershell-build-ps1
          artifact-name: 'HelloWorld.ps1'

# Define a job to be invoked later in a workflow.
# See: https://circleci.com/docs/2.0/configuration-reference/#jobs
jobs:
  # Below is the definition of your job to sign ps1 artifact
  powershell-build-ps1:
    # Create an environment variable
    environment:
        PROJECT_NAME: HelloWorld
        WORKSPACE: /home/circleci/project

    # These next lines define a linux virtual machine executor: https://circleci.com/docs/2.0/executor-types/
    machine: true

    # Working directory for the job on Circle-CI
    working_directory: /home/circleci/project
    
    # Add steps to the job
    # See: https://circleci.com/docs/2.0/configuration-reference/#steps
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - checkout

      # 2) Create Artifact Directory to store signed and unsigned artifact files
      - run:
          name: Create Artifacts Directory
          command: |
            mkdir -p ${WORKSPACE}/artifacts
            mkdir -p ${WORKSPACE}/packages

      # 3) Copy artifact to packages directory
      - run:
          name: Copy Artifact File
          command: | 
            cp powershell/${PROJECT_NAME}.ps1 ${WORKSPACE}/packages/HelloWorld.ps1

      # 4) Persist the artifact to the artifacts directory for signing
      - persist_to_workspace:
          root: .
          paths:
            - packages/*
  
  # Below is the definition of your job to sign artifacts
  codesigner-sign-artifact:
      # Create an environment variable
      environment:
          REGISTRY_USERNAME: bayrakmustafa
          ENVIRONMENT_NAME: TEST
          COMMAND: sign
          WORKSPACE: /home/circleci/project

      # Artifact name for signing
      parameters:
          artifact-name:
            type: string
            default: ''

      # These next lines define a Docker executor: https://circleci.com/docs/2.0/executor-types/
      # You can specify an image from Dockerhub or use one of our Convenience Images from CircleCI's Developer Hub.
      # Be sure to update the Docker image tag below to openjdk version of your application.
      # A list of available CircleCI Docker Convenience Images are available here: https://circleci.com/developer/images/image/cimg/openjdk
      docker:
        - image: cimg/openjdk:17.0.3

      # Working directory for the job on Circle-CI
      working_directory: /home/circleci/project

      # Add steps to the job
      # See: https://circleci.com/docs/2.0/configuration-reference/#steps
      steps:
        # 1) Create Artifact Directory to store signed and unsigned artifact files
        - run:
            name: Create Artifacts Directory
            command: |
              mkdir -p ${WORKSPACE}/artifacts
              mkdir -p ${WORKSPACE}/packages

        # 2) Attach to Workspace in order to access the artifact file
        - attach_workspace:
            at: /home/circleci/project

        # 3) Enable Docker for CodeSigner on Circle-CI
        - setup_remote_docker:
            name: Setup Remote Docker
            version: 19.03.13
            docker_layer_caching: true

        # 4) Pull Codesigner Docker Image From Github Registry
        - run:
            name: Docker Pull Image
            command: |
              docker pull ghcr.io/bayrakmustafa/codesigner:latest
              docker pull alpine:3.4

        # 5) This is the step where the created DLL, JAR, EXE, MSI, PS1 (artifact) files will be signed with CodeSignTool.
        - run:
            name: Sign Artifact File
            command: |
              docker create -v /codesign/packages  --name codesign-in  alpine:3.4 /bin/true
              docker create -v /codesign/artifacts --name codesign-out alpine:3.4 /bin/true
              docker cp ${WORKSPACE}/packages/<< parameters.artifact-name >> codesign-in:/codesign/packages
              docker run -i --rm --dns 8.8.8.8 --network host --volumes-from codesign-in --volumes-from codesign-out -e USERNAME=${USERNAME} -e PASSWORD=${PASSWORD} -e CREDENTIAL_ID=${CREDENTIAL_ID} -e TOTP_SECRET=${TOTP_SECRET} -e ENVIRONMENT_NAME=${ENVIRONMENT_NAME} ghcr.io/bayrakmustafa/codesigner:latest ${COMMAND} -input_file_path=/codesign/packages/<< parameters.artifact-name >> -output_dir_path=/codesign/artifacts
              docker cp codesign-out:/codesign/artifacts/<< parameters.artifact-name >> ${WORKSPACE}/artifacts/<< parameters.artifact-name >>

        # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
        - store_artifacts:
            name: Upload Signed Files
            path: /home/circleci/project/artifacts/<< parameters.artifact-name >>
            destination: << parameters.artifact-name >>
```
<!-- end powershell usage -->

# CodeSignTool Guide

* https://www.ssl.com/guide/esigner-codesigntool-command-guide
* https://www.ssl.com/guide/remote-ev-code-signing-with-esigner/
