# Sign with CodeSignTool

[![GitHub Actions Docker Status](https://github.com/sslcom/ci-images/workflows/Docker%20Image%20CI/badge.svg)](https://github.com/sslcom/ci-images)

CodeSignTool is a secure, privacy-oriented multi-platform Java command line utility for remotely signing Microsoft Authenticode and Java code objects with eSigner EV code signing certificates. Hashes of the files are sent to SSL.com for signing so that the code itself is not sent. This is ideal where sensitive files need to be signed, but should not be sent over the wire for signing. CodeSignTool is also ideal for automated batch processes for high volume signings or integration into existing CI/CD pipeline workflows.

Docker image is used for code signing with GitlabCI. Detailed information: https://github.com/sslcom/ci-images

# Usage (Job)

<!-- start usage -->
```yaml
- stage: Sign
  # When the workflow runs, this is the name that is logged
  displayName: Sign
  jobs:
  - job:
    pool:
      # Run job on Ubuntu VMs
      vmImage: "Ubuntu-latest"
    steps:
      # Created directories for signed artifacts
    - script: mkdir -p ./artifacts && mkdir -p ./packages
      displayName: "Created directories for artifacts and packages"

      # Download to be signed artifact
    - task: DownloadPipelineArtifact@2
      inputs:
        artifact: HelloWorld.dll
        downloadPath: ./packages

      # Install Docker 17.09.0-ce
    - task: DockerInstaller@0
      displayName: Docker Installer
      inputs:
        dockerVersion: 17.09.0-ce
        releaseType: stable

      # Docker Pull CodeSigner Docker Image
    - script: docker pull ghcr.io/sslcom/codesigner:latest
      displayName: 'Docker Pull CodeSigner Docker Image'

      # Sign artifact with CodeSigner docker image
    - script: docker run -i --rm --dns 8.8.8.8 --network host --volume $PWD/packages:/codesign/examples --volume $PWD/artifacts:/codesign/output 
              -e USERNAME=$(USERNAME) -e PASSWORD=$(PASSWORD) -e CREDENTIAL_ID=$(CREDENTIAL_ID) -e TOTP_SECRET=$(TOTP_SECRET) 
              -e ENVIRONMENT_NAME=$(ENVIRONMENT_NAME) ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.dll 
              -output_dir_path=/codesign/output
      displayName: 'Sign artifact with CodeSigner docker image'

      # Save signed artifact for downloading
    - task: PublishBuildArtifacts@1
      displayName: 'Save signed artifact for downloading'
      inputs:
        pathtoPublish: ./artifacts/HelloWorld.dll
        artifactName: HelloWorld.dll
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
# Continuous integration triggers
trigger: 
  - none

# Defines environment variables globally. Job level property overrides global variables
variables: 
  buildConfiguration: 'Release'

# Groups jobs into stages. All jobs in one stage must complete before next stage is executed.
stages:
  - stage: build
    # When the workflow runs, this is the name that is logged
    displayName: Build
    jobs:
      - job:
        pool:
          # Run job on Windows VMs 
          vmImage: 'windows-latest'
        steps:
          # Install Dotnet 6.0.x
        - task: UseDotNet@2
          displayName: 'Install .NET Core SDK'
          inputs:
            version: '6.0.x'
            performMultiLevelLookup: true
            includePreviewVersions: true
        
           # Restore Dotnet project
        - task: DotNetCoreCLI@2
          displayName: Restore
          inputs:
            commmand: 'restore'

          # Build dotnet project with Release configuration
        - task: DotNetCoreCLI@2
          displayName: Build
          inputs:
            commmand: build
            projects: HelloWorld.csproj
            arguments: '--configuration $(buildConfiguration)'

          # Created directories for signed artifacts
        - powershell: New-Item -ItemType Directory -Path ./artifacts
          displayName: 'Created directories for artifacts'

          # Created directories for signed packages
        - powershell: New-Item -ItemType Directory -Path ./packages
          displayName: 'Created directories for packages'

          # Copy artifact to be signing path
        - powershell: Copy-Item ./bin/Release/netcoreapp3.1/HelloWorld-0.0.1.dll -Destination ./packages/HelloWorld.dll
          displayName: "Copy built artifacts to packages directory"

          # Save artifact in order to use signing job
        - task: PublishBuildArtifacts@1
          displayName: 'Save to be signed artifact for downloading'
          inputs:
            pathtoPublish: ./packages/HelloWorld.dll
            artifactName: HelloWorld.dll

  - stage: Sign
    # When the workflow runs, this is the name that is logged
    displayName: Sign
    jobs:
    - job:
      pool:
        # Run job on Ubuntu VMs
        vmImage: "Ubuntu-latest"
      steps:
        # Created directories for signed artifacts
      - script: mkdir -p ./artifacts && mkdir -p ./packages
        displayName: "Created directories for artifacts and packages"

        # Download to be signed artifact
      - task: DownloadPipelineArtifact@2
        inputs:
          artifact: HelloWorld.dll
          downloadPath: ./packages

        # Install Docker 17.09.0-ce
      - task: DockerInstaller@0
        displayName: Docker Installer
        inputs:
          dockerVersion: 17.09.0-ce
          releaseType: stable

        # Docker Pull CodeSigner Docker Image
      - script: docker pull ghcr.io/sslcom/codesigner:latest
        displayName: 'Docker Pull CodeSigner Docker Image'

        # Sign artifact with CodeSigner docker image
      - script: docker run -i --rm --dns 8.8.8.8 --network host --volume $PWD/packages:/codesign/examples --volume $PWD/artifacts:/codesign/output 
                -e USERNAME=$(USERNAME) -e PASSWORD=$(PASSWORD) -e CREDENTIAL_ID=$(CREDENTIAL_ID) -e TOTP_SECRET=$(TOTP_SECRET) 
                -e ENVIRONMENT_NAME=$(ENVIRONMENT_NAME) ghcr.io/sslcom/codesigner:latest sign -input_file_path=/codesign/examples/HelloWorld.dll 
                -output_dir_path=/codesign/output
        displayName: 'Sign artifact with CodeSigner docker image'

        # Save signed artifact for downloading
      - task: PublishBuildArtifacts@1
        displayName: 'Save signed artifact for downloading'
        inputs:
          pathtoPublish: ./artifacts/HelloWorld.dll
          artifactName: HelloWorld.dll
```
<!-- end dotnet usage -->

# CodeSignTool Guide

* https://www.ssl.com/guide/esigner-codesigntool-command-guide
* https://www.ssl.com/guide/remote-ev-code-signing-with-esigner/
