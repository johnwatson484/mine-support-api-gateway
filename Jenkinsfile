@Library('defra-library@0.0.3')
import uk.gov.defra.ffc.DefraUtils
def defraUtils = new DefraUtils()

def registry = '562955126301.dkr.ecr.eu-west-2.amazonaws.com'
def regCredsId = 'ecr:eu-west-2:ecr-user'
def kubeCredsId = 'awskubeconfig002'
def imageName = 'ffc-demo-api-gateway'
def repoName = 'ffc-demo-api-gateway'
def pr = ''
def mergedPrNo = ''
def containerTag = ''

node {
  checkout scm

  try {
    stage('Set branch, PR, and containerTag variables') {
      (pr, containerTag, mergedPrNo) = defraUtils.getVariables(repoName)
      defraUtils.setGithubStatusPending()
    }
    stage('Build test image') {
      defraUtils.buildTestImage(imageName, BUILD_NUMBER)
    }
    stage('Run tests') {
      defraUtils.runTests(imageName, BUILD_NUMBER)
    }
    stage('Push container image') {
      defraUtils.buildAndPushContainerImage(regCredsId, registry, imageName, containerTag)
    }
    if (pr != '') {
      stage('Helm install') {
        def extraCommands = "--values ./helm/$repoName/jenkins-aws.yaml"
        defraUtils.deployChart(kubeCredsId, registry, imageName, containerTag, extraCommands)
      }
    }
    if (pr == '') {
      stage('Publish chart') {
        defraUtils.publishChart(registry, imageName, containerTag)
      }
      stage('Trigger Deployment') {
        withCredentials([
          string(credentialsId: 'JenkinsDeployUrl', variable: 'jenkinsDeployUrl'),
          string(credentialsId: 'ffc-demo-api-gateway-deploy-token', variable: 'jenkinsToken')
        ]) {
          defraUtils.triggerDeploy(jenkinsDeployUrl, 'ffc-demo-api-gateway-deploy', jenkinsToken, ['chartVersion':'1.0.0'])
        }
      }
    }
    if (mergedPrNo != '') {
      stage('Remove merged PR') {
        defraUtils.undeployChart(kubeCredsId, imageName, mergedPrNo)
      }
    }
    defraUtils.setGithubStatusSuccess()
  } catch(e) {
    defraUtils.setGithubStatusFailure(e.message)
    throw error
  }
}
