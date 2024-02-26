```yaml

trigger:
- none

pool:
  name: Aditya
  demands: agent.name -equals agent-1

stages:
- stage: Compile
  displayName: 'Compile Stage'
  jobs:
  - job: CompileJob
    displayName: 'Compile Job'
    pool:
      name: Aditya
      demands: agent.name -equals agent-1
    steps:
    - script: mvn compile
      displayName: 'Compile-Step'

- stage: Trivy_FS_Scan
  displayName: 'Trivy FS Stage'
  jobs:
  - job: TrivyFSJob
    displayName: 'TrivyFS Job'
    pool:
      name: Aditya
      demands: agent.name -equals agent-1
    steps:
    - script: trivy fs --format table -o trivy-fs-report.html .
      displayName: 'TrivyFS-Scan-Step'

- stage: SonarQube_Scan
  displayName: 'SonarQube Stage'
  jobs:
  - job: SonarQubeJob
    displayName: 'SonarQube Job'
    pool:
      name: Aditya
      demands: agent.name -equals agent-1
    steps:
    - task: SonarQubePrepare@5
      inputs:
        SonarQube: 'sonar-svc'
        scannerMode: 'CLI'
        configMode: 'manual'
        cliProjectKey: 'screte-santa'
        cliProjectName: 'screte-santa'
        cliSources: '.'
        extraProperties: |
          # Additional properties that will be passed to the scanner, 
          # Put one key=value per line, example:
          # sonar.exclusions=**/*.bin
          sonar.java.binaries=.
    - task: SonarQubeAnalyze@5
      inputs:
        jdkversion: 'JAVA_HOME'

- stage: Build
  displayName: 'Build Stage'
  jobs:
  - job: BuildJob
    displayName: 'Build Job'
    pool:
      name: Aditya
      demands: agent.name -equals agent-1
    steps:
    - script: mvn package
      displayName: 'Build-Package-Step'

- stage: Docker
  displayName: 'Docker Stage'
  jobs:
  - job: DockerJob
    displayName: 'Docker Job'
    pool:
      name: Aditya
      demands: agent.name -equals agent-1
    steps:
    - script: mvn package
      displayName: 'Build-Package-Step'
    - task: Docker@2
      inputs:
        containerRegistry: 'docker-svc'
        repository: 'adijaiswal/santa'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        tags: 'latest'

- stage: Trivy_Image_Scan
  displayName: 'Trivy Image Stage'
  jobs:
  - job: TrivyImageJob
    displayName: 'Trivy Image Job'
    pool:
      name: Aditya
      demands: agent.name -equals agent-1
    steps:
    - script: trivy image --format table -o trivy-fs-report.html adijaiswal/santa:latest
      displayName: 'Trivy-Image-Scan-Step'

- stage: K8_Deploy
  displayName: 'K8_Deploy Stage'
  jobs:
  - job: TK8_DeployJob
    displayName: 'K8_Deploy Job'
    pool:
      name: Aditya
      demands: agent.name -equals agent-1
    steps:
    - task: KubectlInstaller@0
      inputs:
        kubectlVersion: 'latest'
    - task: Kubernetes@1
      inputs:
        connectionType: 'Kubernetes Service Connection'
        kubernetesServiceEndpoint: 'k8-service-connection'
        namespace: 'default'
        command: 'apply'
        useConfigurationFile: true
        configuration: 'k8-dep-svc.yml'
        secretType: 'dockerRegistry'
        containerRegistryType: 'Container Registry'
        dockerRegistryEndpoint: 'docker-svc'
        forceUpdate: false
