# Sign with CodeSignTool

[![GitHub Actions Status](https://github.com/bayrakmustafa/codesigner-samples/workflows/CodeSignTool/badge.svg)](https://github.com/bayrakmustafa/codesigner-samples)
[![Build Status](https://img.shields.io/circleci/build/github/bayrakmustafa/codesigner-samples/main.svg?style=plastic&label=Circle%20CI&branch=main)](https://circleci.com/gh/bayrakmustafa/codesigner-samples/tree/main)
[![GitlabCI](https://gitlab.com/mustafabayrak/codesigner-samples/badges/main/pipeline.svg?key_text=Gitlab%20CI)](https://gitlab.com/mustafabayrak/codesigner-samples/-/commits/main)
[![Build Status](https://img.shields.io/travis/bayrakmustafa/codesigner-samples.svg?style=plastic&label=Travis%20CI&branch=main)](https://app.travis-ci.com/bayrakmustafa/codesigner-samples)

CodeSignTool is a secure, privacy-oriented multi-platform Java command line utility for remotely signing Microsoft Authenticode and Java code objects with eSigner EV code signing certificates. Hashes of the files are sent to SSL.com for signing so that the code itself is not sent. This is ideal where sensitive files need to be signed, but should not be sent over the wire for signing. CodeSignTool is also ideal for automated batch processes for high volume signings or integration into existing CI/CD pipeline workflows.

This action provides the sign artifacts with CodeSignTool.

`ES_USERNAME`, `ES_PASSWORD`, `CRENDENTIAL_ID` and `ES_TOTP_SECRET` values to be used in CodeSignTool can be stored in the Actons Secrets area of Github.
Detailed Information: https://docs.github.com/en/actions/security-guides/encrypted-secrets

# Usage

<!-- start usage -->
```yaml
- name: Sign Artifact with CodeSignTool
  uses: sslcom/actions-codesigner@develop
  with:
    # CodeSignTool Commands:
    # - get_credential_ids: Output the list of eSigner credential IDs associated with a particular user.
    # - credential_info: Output key and certificate information related to a credential ID.
    # - sign: Sign and timestamp code object.
    # - batch_sign: Sign and timestamp multiple code objects with one OTP.
    # - hash: Pre-compute hash(es) for later use with batch_hash_sign command.
    # - batch_sign_hash: Sign hash(es) pre-computed with hash command.
    command: sign

    # SSL.com account username..
    username: ${{secrets.ES_USERNAME}}

    # SSL.com account password.
    password: ${{secrets.ES_PASSWORD}}

    # Credential ID for signing certificate.
    credential_id: ${{secrets.CREDENTIAL_ID}}

    # OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
    totp_secret: ${{secrets.ES_TOTP_SECRET}}

    # Path of code object to be signed.
    # Supported File Types: acm, ax, bin, cab, cpl, dll, drv, efi, exe, mui, ocx, scr, sys, tsp, msi, ps1, ps1xml, js, vbs, wsf, jar
    file_path: ${GITHUB_WORKSPACE}/test/src/build/HelloWorld.jar

    # Directory where signed code object(s) will be written.
    output_path: ${GITHUB_WORKSPACE}/artifacts
```
<!-- end usage -->

### Inputs

* `username`: SSL.com account username. (**Required**)

* `password`: SSL.com account password (**Required**)

* `credential_id`: Credential ID for signing certificate. If credential_id is omitted and the user has only one eSigner code signing certificate, CodeSignTool will default to that. If the user has more than one code signing certificate, this parameter is mandatory. (**Required**)

* `totp_secret`: OAuth TOTP Secret. You can access detailed information on https://www.ssl.com/how-to/automate-esigner-ev-code-signing (**Required**)

* `file_path`: Path of code object to be signed. (**Required**)

* `output_path`: Directory where signed code object(s) will be written. If output_path is omitted, the file specified in -file_path will be overwritten with the signed file.

# Examples

## Dotnet Code DLL Signing Example Workflow

<!-- start dotnet usage -->
```yml
# The name of the workflow.
name: (DLL) Dotnet Core Build and Sign

# Trigger this workflow on a push
on: push

# Create an environment variable
env:
  PROJECT_NAME: HelloWorld
  PROJECT_VERSION: 0.0.1
  DOTNET_VERSION: 3.1

# Defines a single job named "codesigner-dotnet-dll"
jobs:
  codesigner-dotnet-dll:
    # Run job on Ubuntu Runner
    runs-on: ubuntu-latest
    # When the workflow runs, this is the name that is logged
    name: CodeSigner on Dotnet
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - name: Checkout Repository
        uses: actions/checkout@v2

      # 2) Set up the .NET CLI environment for the workflow to use.
      - name: Setup Dotnet Core
        uses: actions/setup-dotnet@v2
        with:
          dotnet-version: '3.1.x'

      # 3) Create Artifact Directory to store signed and unsigned artifact files
      - name: Create Artifacts Directory
        shell: bash
        run: |
          mkdir ${GITHUB_WORKSPACE}/artifacts
          mkdir ${GITHUB_WORKSPACE}/packages

      # 4) Build a dotnet project or solution and all of its dependencies.
      #    After it has been created dll or exe file, copy to 'packages' folder for siging
      - name: Compile Dotnet Library
        shell: bash
        run: |
          dotnet build dotnet/${{env.PROJECT_NAME}}.csproj -c Release
          cp dotnet/bin/Release/netcoreapp${{env.DOTNET_VERSION}}/${{env.PROJECT_NAME}}-${{env.PROJECT_VERSION}}.dll ${GITHUB_WORKSPACE}/packages/${{env.PROJECT_NAME}}.dll

      # 5) This is the step where the created DLL or EXE (artifact) files will be signed with CodeSignTool.
      - name: Sign Artifact with CodeSignTool
        uses: bayrakmustafa/actions-codesigner@develop
        with:
          # Sign and timestamp code object.
          command: sign
          # SSL.com account username.
          username: ${{secrets.ES_USERNAME}}
          # SSL.com account password.
          password: ${{secrets.ES_PASSWORD}}
          # Credential ID for signing certificate.
          credential_id: ${{secrets.CREDENTIAL_ID}}
          # OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
          totp_secret: ${{secrets.ES_TOTP_SECRET}}
          # Path of code object to be signed. (DLL, JAR, EXE, MSI files vb... )
          file_path: ${GITHUB_WORKSPACE}/packages/${{env.PROJECT_NAME}}.dll
          # Directory where signed code object(s) will be written.
          output_path: ${GITHUB_WORKSPACE}/artifacts

        # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
      - name: Upload Signed Files
        uses: actions/upload-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.dll
          path: ./artifacts/${{env.PROJECT_NAME}}.dll
```
<!-- end dotnet usage -->

## Java Code (Maven) JAR Signing Example Workflow

<!-- start maven usage -->
```yml
# The name of the workflow.
name: (JAR) Maven Build and Sign

# Trigger this workflow on a push
on: push

# Create an environment variable
env:
  PROJECT_NAME: HelloWorld
  PROJECT_VERSION: 0.0.1
  MAVEN_VERSION: 3.8.5
  JAVA_VERSION: 17

# Defines a single job named "codesigner-maven-jar"
jobs:
  codesigner-maven-jar:
    # Run job on Ubuntu Runner
    runs-on: ubuntu-latest
    # When the workflow runs, this is the name that is logged
    name: CodeSigner on Java with Maven
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - name: Checkout Repository
        uses: actions/checkout@v2

      # 2) Set up the Java and Maven environment for the workflow to use.
      - name: Setup Java and Maven
        uses: s4u/setup-maven-action@v1.3.1
        with:
          java-version: '${{env.JAVA_VERSION}}'
          maven-version: '${{env.MAVEN_VERSION}}'

      # 3) Create Artifact Directory to store signed and unsigned artifact files
      - name: Create Artifacts Directory
        shell: bash
        run: |
          mkdir ${GITHUB_WORKSPACE}/artifacts
          mkdir ${GITHUB_WORKSPACE}/packages

      # 4) Build a maven project or solution and all of its dependencies.
      #    After it has been created jar file, copy to 'packages' folder for siging
      - name: Compile Java Library with Maven
        shell: bash
        run: |
          mvn clean install -f java/pom.xml
          cp java/target/${{env.PROJECT_NAME}}-${{env.PROJECT_VERSION}}.jar ${GITHUB_WORKSPACE}/packages/${{env.PROJECT_NAME}}.jar

      # 5) This is the step where the created JAR (artifact) files will be signed with CodeSignTool.
      - name: Sign Artifact with CodeSignTool
        uses: bayrakmustafa/actions-codesigner@develop
        with:
          # Sign and timestamp code object.
          command: sign
          # SSL.com account username.
          username: ${{secrets.ES_USERNAME}}
          # SSL.com account password.
          password: ${{secrets.ES_PASSWORD}}
          # Credential ID for signing certificate.
          credential_id: ${{secrets.CREDENTIAL_ID}}
          # OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
          totp_secret: ${{secrets.ES_TOTP_SECRET}}
          # Path of code object to be signed. (DLL, JAR, EXE, MSI files vb... )
          file_path: ${GITHUB_WORKSPACE}/packages/${{env.PROJECT_NAME}}.jar
          # Directory where signed code object(s) will be written.
          output_path: ${GITHUB_WORKSPACE}/artifacts

        # 6) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
      - name: Upload Signed Files
        uses: actions/upload-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.jar
          path: ./artifacts/${{env.PROJECT_NAME}}.jar
```
<!-- end maven usage -->

## Java Code (Gradle) JAR Signing Example Workflow

<!-- start gradle usage -->
```yml
# The name of the workflow.
name: (JAR) Gradle Build and Sign

# Trigger this workflow on a push
on: push

# Create an environment variable
env:
  PROJECT_NAME: HelloWorld
  PROJECT_VERSION: 0.0.1
  GRADLE_VERSION: 7.3
  JAVA_VERSION: 17

# Defines a single job named "codesigner-gradle-jar"
jobs:
  codesigner-gradle-jar:
    # Run job on Ubuntu Runner
    runs-on: ubuntu-latest
    # When the workflow runs, this is the name that is logged
    name: CodeSigner on Java with Gradle
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - name: Checkout Repository
        uses: actions/checkout@v2

      # 2) Set up the Java environment for the workflow to use.
      - uses: actions/setup-java@v2
        with:
          distribution: zulu
          java-version: ${{env.JAVA_VERSION}}

      # 3) Set up the Gradle environment for the workflow to use.
      - name: Setup Gradle
        uses: gradle/gradle-build-action@v2
        with:
          gradle-version: ${{env.GRADLE_VERSION}}

      # 4) Create Artifact Directory to store signed and unsigned artifact files
      - name: Create Artifacts Directory
        shell: bash
        run: |
          mkdir ${GITHUB_WORKSPACE}/artifacts
          mkdir ${GITHUB_WORKSPACE}/packages

      # 5) Build a gradle project or solution and all of its dependencies.
      #    After it has been created jar file, copy to 'packages' folder for siging
      - name: Compile Java Library with Gradle
        shell: bash
        run: |
          gradle clean build -p java -PsetupType=jar
          cp java/build/libs/${{env.PROJECT_NAME}}-${{env.PROJECT_VERSION}}.jar ${GITHUB_WORKSPACE}/packages/${{env.PROJECT_NAME}}.jar

      # 6) This is the step where the created JAR (artifact) files will be signed with CodeSignTool.
      - name: Sign Artifact with CodeSignTool
        uses: bayrakmustafa/actions-codesigner@develop
        with:
          # Sign and timestamp code object.
          command: sign
          # SSL.com account username.
          username: ${{secrets.ES_USERNAME}}
          # SSL.com account password.
          password: ${{secrets.ES_PASSWORD}}
          # Credential ID for signing certificate.
          credential_id: ${{secrets.CREDENTIAL_ID}}
          # OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
          totp_secret: ${{secrets.ES_TOTP_SECRET}}
          # Path of code object to be signed. (DLL, JAR, EXE, MSI files vb... )
          file_path: ${GITHUB_WORKSPACE}/packages/${{env.PROJECT_NAME}}.jar
          # Directory where signed code object(s) will be written.
          output_path: ${GITHUB_WORKSPACE}/artifacts

        # 7) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
      - name: Upload Signed Files
        uses: actions/upload-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.jar
          path: ./artifacts/${{env.PROJECT_NAME}}.jar
```
<!-- end gradle usage -->

## Java Code (Gradle) MSI Signing Example Workflow

<!-- start gradle usage -->
```yml
# The name of the workflow.
name: (MSI) Gradle Build and Sign

# Trigger this workflow on a push
on: push

# Create an environment variable
env:
  PROJECT_NAME: HelloWorld
  PROJECT_VERSION: 0.0.1
  GRADLE_VERSION: 7.3
  JAVA_VERSION: 17

jobs:
  # Defines job named "build-gradle-msi"
  build-gradle-msi:
    # Run job on Windows Runner
    runs-on: windows-latest
    # When the workflow runs, this is the name that is logged
    name: Build MSI File with Gradle
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - name: Checkout Repository
        uses: actions/checkout@v2

      # 2) Set up the Java environment for the workflow to use.
      - uses: actions/setup-java@v2
        with:
          distribution: zulu
          java-version: ${{env.JAVA_VERSION}}

      # 3) Set up the Gradle environment for the workflow to use.
      - name: Setup Gradle
        uses: gradle/gradle-build-action@v2
        with:
          gradle-version: ${{env.GRADLE_VERSION}}

      # 4) Install WIX for MSI Setup File
      - name: Windows Specific Setup
        run: |
          curl -OLS https://github.com/wixtoolset/wix3/releases/download/wix3111rtm/wix311.exe
          .\wix311.exe /install /quiet /norestart

      # 5) Add WIX to environment variable
      - name: Add path to PATH environment variable
        uses: myci-actions/export-env-var-powershell@1
        with:
          name: PATH
          value: $env:PATH;C:\Program Files (x86)\WiX Toolset v3.11\bin

      # 6) Create Artifact Directory to store signed and unsigned artifact files
      - name: Create Artifacts Directory
        shell: bash
        run: |
          mkdir ${GITHUB_WORKSPACE}/artifacts
          mkdir ${GITHUB_WORKSPACE}/packages

      # 7) Build a gradle project or solution and all of its dependencies.
      #    After it has been created jar file, copy to 'packages' folder for siging
      - name: Compile Java Library with Gradle
        shell: bash
        run: |
          gradle build jpackage -x test --warning-mode all -p java -PsetupType=msi
          cp java/build/release/windows/*.msi ${GITHUB_WORKSPACE}/packages/${{env.PROJECT_NAME}}.msi

      # 8) Save the MSI file to the artifacts directory for codesigning tool
      - name: Upload Setup File
        id: upload-installer-windows
        uses: actions/upload-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.msi
          path: ./packages/${{env.PROJECT_NAME}}.msi

  # Defines job named "codesigner-msi"
  codesigner-msi:
    # Codesigner job only work with linux runner
    runs-on: ubuntu-latest
    # Run this job after build-gradle-msi job is finished
    needs: [build-gradle-msi]
    # When the workflow runs, this is the name that is logged
    name: CodeSigner on MSI File
    steps:
      # 1) Download created msi file from Github Artifacts
      - name: Download Windows Installer File
        uses: actions/download-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.msi

      # 2) Create Artifact Directory to store signed and unsigned artifact files
      - name: Create Artifacts Directory
        shell: bash
        run: |
          mkdir ${GITHUB_WORKSPACE}/artifacts

      # 3) This is the step where the created JAR (artifact) files will be signed with CodeSignTool.
      - name: Sign Artifact with CodeSignTool
        uses: bayrakmustafa/actions-codesigner@develop
        with:
          # Sign and timestamp code object.
          command: sign
          # SSL.com account username.
          username: ${{secrets.ES_USERNAME}}
          # SSL.com account password.
          password: ${{secrets.ES_PASSWORD}}
          # Credential ID for signing certificate.
          credential_id: ${{secrets.CREDENTIAL_ID}}
          # OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
          totp_secret: ${{secrets.ES_TOTP_SECRET}}
          # Path of code object to be signed. (DLL, JAR, EXE, MSI files vb... )
          file_path: ${GITHUB_WORKSPACE}/${{env.PROJECT_NAME}}.msi
          # Directory where signed code object(s) will be written.
          output_path: ${GITHUB_WORKSPACE}/artifacts

      # 4) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
      - name: Upload Signed Files
        uses: actions/upload-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.msi
          path: ./artifacts/${{env.PROJECT_NAME}}.msi
```
<!-- end gradle usage -->

## Java Code (Gradle) EXE Signing Example Workflow

<!-- start gradle usage -->
```yml
# The name of the workflow.
name: (EXE) Gradle Build and Sign

# Trigger this workflow on a push
on: push

# Create an environment variable
env:
  PROJECT_NAME: HelloWorld
  PROJECT_VERSION: 0.0.1
  GRADLE_VERSION: 7.3
  JAVA_VERSION: 17

jobs:
  # Defines job named "build-gradle-exe"
  build-gradle-exe:
    # Run job on Windows Runner
    runs-on: windows-latest
    # When the workflow runs, this is the name that is logged
    name: Build exe File with Gradle
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - name: Checkout Repository
        uses: actions/checkout@v2

      # 2) Set up the Java environment for the workflow to use.
      - uses: actions/setup-java@v2
        with:
          distribution: zulu
          java-version: ${{env.JAVA_VERSION}}

      # 3) Set up the Gradle environment for the workflow to use.
      - name: Setup Gradle
        uses: gradle/gradle-build-action@v2
        with:
          gradle-version: ${{env.GRADLE_VERSION}}

      # 4) Install WIX for exe Setup File
      - name: Windows Specific Setup
        run: |
          curl -OLS https://github.com/wixtoolset/wix3/releases/download/wix3111rtm/wix311.exe
          .\wix311.exe /install /quiet /norestart

      # 5) Add WIX to environment variable
      - name: Add path to PATH environment variable
        uses: myci-actions/export-env-var-powershell@1
        with:
          name: PATH
          value: $env:PATH;C:\Program Files (x86)\WiX Toolset v3.11\bin

      # 6) Create Artifact Directory to store signed and unsigned artifact files
      - name: Create Artifacts Directory
        shell: bash
        run: |
          mkdir ${GITHUB_WORKSPACE}/artifacts
          mkdir ${GITHUB_WORKSPACE}/packages

      # 7) Build a gradle project or solution and all of its dependencies.
      #    After it has been created jar file, copy to 'packages' folder for siging
      - name: Compile Java Library with Gradle
        shell: bash
        run: |
          gradle build jpackage -x test --warning-mode all -p java -PsetupType=exe
          cp java/build/release/windows/*.exe ${GITHUB_WORKSPACE}/packages/${{env.PROJECT_NAME}}.exe

      # 8) Save the exe file to the artifacts directory for codesigning tool
      - name: Upload Setup File
        id: upload-installer-windows
        uses: actions/upload-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.exe
          path: ./packages/${{env.PROJECT_NAME}}.exe

  # Defines job named "codesigner-exe"
  codesigner-exe:
    # Codesigner job only work with linux runner
    runs-on: ubuntu-latest
    # Run this job after build-gradle-exe job is finished
    needs: [build-gradle-exe]
    # When the workflow runs, this is the name that is logged
    name: CodeSigner on exe File
    steps:
      # 1) Download created exe file from Github Artifacts
      - name: Download Windows Installer File
        uses: actions/download-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.exe

      # 2) Create Artifact Directory to store signed and unsigned artifact files
      - name: Create Artifacts Directory
        shell: bash
        run: |
          mkdir ${GITHUB_WORKSPACE}/artifacts

      # 3) This is the step where the created JAR (artifact) files will be signed with CodeSignTool.
      - name: Sign Artifact with CodeSignTool
        uses: bayrakmustafa/actions-codesigner@develop
        with:
          # Sign and timestamp code object.
          command: sign
          # SSL.com account username.
          username: ${{secrets.ES_USERNAME}}
          # SSL.com account password.
          password: ${{secrets.ES_PASSWORD}}
          # Credential ID for signing certificate.
          credential_id: ${{secrets.CREDENTIAL_ID}}
          # OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
          totp_secret: ${{secrets.ES_TOTP_SECRET}}
          # Path of code object to be signed. (DLL, JAR, EXE, exe files vb... )
          file_path: ${GITHUB_WORKSPACE}/${{env.PROJECT_NAME}}.exe
          # Directory where signed code object(s) will be written.
          output_path: ${GITHUB_WORKSPACE}/artifacts

      # 4) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
      - name: Upload Signed Files
        uses: actions/upload-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.exe
          path: ./artifacts/${{env.PROJECT_NAME}}.exe
```
<!-- end gradle usage -->

## Powershell PS1 Signing Example Workflow

<!-- start powershell usage -->
```yml
# The name of the workflow.
name: (PS1) Poweshell Script Signing

# Trigger this workflow on a push
on: push

# Create an environment variable
env:
  PROJECT_NAME: HelloWorld

# Defines a single job named "codesigner-powershell-ps1"
jobs:
  codesigner-powershell-ps1:
    # Run job on Ubuntu Runner
    runs-on: ubuntu-latest
    # When the workflow runs, this is the name that is logged
    name: CodeSigner on Powershell
    steps:
      # 1) Check out the source code so that the workflow can access it.
      - name: Checkout Repository
        uses: actions/checkout@v2

      # 2) Create Artifact Directory to store signed and unsigned artifact files
      - name: Create Artifacts Directory
        shell: bash
        run: |
          mkdir ${GITHUB_WORKSPACE}/artifacts

      # 3) This is the step PS1 file will be signed with CodeSignTool.
      - name: Sign Artifact with CodeSignTool
        uses: bayrakmustafa/actions-codesigner@develop
        with:
          # Sign and timestamp code object.
          command: sign
          # SSL.com account username.
          username: ${{secrets.ES_USERNAME}}
          # SSL.com account password.
          password: ${{secrets.ES_PASSWORD}}
          # Credential ID for signing certificate.
          credential_id: ${{secrets.CREDENTIAL_ID}}
          # OAuth TOTP Secret (https://www.ssl.com/how-to/automate-esigner-ev-code-signing)
          totp_secret: ${{secrets.ES_TOTP_SECRET}}
          # Path of code object to be signed. (DLL, JAR, EXE, MSI files vb... )
          file_path: ${GITHUB_WORKSPACE}/powershell/${{env.PROJECT_NAME}}.ps1
          # Directory where signed code object(s) will be written.
          output_path: ${GITHUB_WORKSPACE}/artifacts

        # 4) This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete
      - name: Upload Signed Files
        uses: actions/upload-artifact@v2
        with:
          name: ${{env.PROJECT_NAME}}.ps1
          path: ./artifacts/${{env.PROJECT_NAME}}.ps1
```
<!-- end powershell usage -->

# CodeSignTool Guide

* https://www.ssl.com/guide/esigner-codesigntool-command-guide
* https://www.ssl.com/guide/remote-ev-code-signing-with-esigner/
