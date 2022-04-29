import hudson.model.*
import java.text.SimpleDateFormat
import java.util.regex.Matcher

properties([disableConcurrentBuilds(), buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '30', numToKeepStr: '')), \
  parameters([string(defaultValue: '2022.2.0', description: 'Develop version being built', name: 'VERSION', trim: true)])

])

if (currentBuild.rawBuild.getCauses().toString().contains('BranchIndexingCause')) {
    print "INFO: Build skipped due to trigger being 'Branch Indexing'"
    return
}

def sonarqube_server = "SparkSonar"
def sonarscanner_tool = "SonarMSScanner"
def develop = false
def master = false
def staging= false
def environment = ""
def bucket = ""

def sqscanner = "C:/Users/jenkins/.dotnet/tools/dotnet-sonarscanner.exe"

def azure_sp_credentials
def azure_storage_account
def foundationdi_installer_filename = ""
def bloblist_prefix = "${env.JOB_NAME}".replaceAll('/','-').replaceAll(' ','-').trim() + "-${environment}"


if ("${env.BRANCH_NAME}" == "develop") {
      develop = true
    environment = "dev"
    bucket = "foundationdi"
    azure_sp_credentials = "azure-da-dev"
    azure_storage_account = "sparkcognitionaif"
} else if ("${env.BRANCH_NAME}" == "master") {
    master = true
    environment = "prod"
    bucket = "foundationdi"
    azure_sp_credentials = "azure-da-dev"
    azure_storage_account = "sparkcognitionaif"
} else if ("${env.BRANCH_NAME}" == "stage") {
    staging = true
    environment = "staging"
    bucket = "foundationdi"
    azure_sp_credentials = "azure-da-dev"
    azure_storage_account = "sparkcognitionaif"
} else {
    environment = "${env.BRANCH_NAME}"
    develop = false
    bucket = "foundationdi"
    azure_sp_credentials = "azure-da-dev"
    azure_storage_account = "sparkcognitionaif"
}

def stage(name, execute, block) {
    return stage(name, execute ? block : {echo "Skipped stage $name"})
}

def DevVersion(version) {
    return version.substring(0, version.length()-2)
}
node('da-windows') {
    try {
        stage('Job Info') {
            echo "${env.JOB_NAME} build number ${env.BUILD_NUMBER}"
            echo "Building ${env.BRANCH_NAME} for ${environment} environment"
            slackSend channel: 'aifoundation', color: 'good', message: "Started ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", tokenCredentialId: 'slack-token'
        }

        stage('Checkinout and Tagin') {
            cleanWs()
            checkout([
                $class: 'GitSCM',
                branches: scm.branches,
                doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
                extensions: [[$class: 'CloneOption', noTags: false, shallow: false, depth: 0, reference: '']],
                userRemoteConfigs: scm.userRemoteConfigs,
            ])

            if (master) {
                def git_tag = bat returnStdout: true, script: "git describe --tags --always"
                def v = "${git_tag.trim()}".split(' ')[6];
                def short_version = "${v}".split("-")[0]
                version = "${short_version.trim()}"
                echo "Building master - ${version}"
            }
            else
            {
                version = DevVersion("${params.VERSION}")+".${BUILD_NUMBER}"
                echo "Building ${version}"
            }
        }
        stage("Register AI") {
            withCredentials([string(credentialsId: 'advanced_installer_license_key', variable: 'advanced_installer_license_key')]) {
                bat(""""C:/Program Files (x86)/Caphyon/Advanced Installer 19.2/bin/x86/AdvancedInstaller.com" /register ${advanced_installer_license_key}""")
            }
        }
        stage('Buildin sie packages') {
            bat 'dotnet publish FoundationDI.lib.Connector.AWS/FoundationDI.lib.Connector.AWS.csproj -c Release --output ./Release/net6.0-windows/'
            bat 'dotnet publish FoundationDI.lib.Connector.OSIPI/FoundationDI.lib.Connector.OSIPI.csproj -c Release --output ./Release/net6.0-windows/'
            bat 'dotnet publish FoundationDI.lib.Connector.ODBC/FoundationDI.lib.Connector.ODBC.csproj -c Release --output ./Release/net6.0-windows/'
            bat 'dotnet publish FoundationDI.Service.Windows/FoundationDI.Service.Windows.csproj -c Release --self-contained true -r win-x64 --output ./Release/net6.0-windows/'
            bat 'dotnet publish FoundationIngest.App.CLI/FoundationIngest.App.CLI.csproj -c Release --self-contained true -r win-x64 --output ./Release/net6.0-windows/'
            bat 'dotnet build FoundationIngest.App.GUI/FoundationIngest.App.GUI.csproj -c Release --self-contained true -r win-x64 --output ./Release/net6.0-windows/'
            foundationdi_installer_filename = "${environment}_foundationdi_${version}.msi"

            bat """"C:/Program Files (x86)/Caphyon/Advanced Installer 19.2/bin/x86/AdvancedInstaller.com" /edit Installer/Installer.aip /SetVersion ${version}"""

            echo "Build ${version} Installer"
            bat """ "C:/Program Files (x86)/Caphyon/Advanced Installer 19.2/bin/x86/AdvancedInstaller.com" /build Installer/Installer.aip"""

            bat "mkdir artifacts"
            bat "move Installer\\Installer-SetupFiles\\Installer.msi artifacts\\${foundationdi_installer_filename}"
        }

        stage('Publish sie artifacts') {
            milestone label: 'Publish'
                withCredentials([
                    file(credentialsId: "${azure_sp_credentials}-cert", variable: 'CERT'),
                    azureServicePrincipal(azure_sp_credentials),
                ]) {
                    bat """"C:/Program Files (x86)/Microsoft SDKs/Azure/CLI2/wbin/az.cmd" login --service-principal --username ${AZURE_CLIENT_ID} --password "$CERT" --tenant ${AZURE_TENANT_ID}"""
                    bat """"C:/Program Files (x86)/Microsoft SDKs/Azure/CLI2/wbin/az.cmd" extension add --name storage-preview"""
                    bat """"C:/Program Files (x86)/Microsoft SDKs/Azure/CLI2/wbin/az.cmd" storage blob directory list --auth-mode login --account-name ${azure_storage_account} -c ${bucket} -d ${environment} --prefix "latest/" --query "[].{name:name}" --output tsv > ${bloblist_prefix}-bloblist.txt"""
                    bat """
                    for /f %%i in ("${bloblist_prefix}-bloblist.txt") do set size=%%~zi
                    if %size% gtr 0 for /F "tokens=*" %%b in (${bloblist_prefix}-bloblist.txt) do (echo %%~nxb>>${bloblist_prefix}-bloblist-updated.txt)
                    if exist "${bloblist_prefix}-bloblist-updated.txt" for /F "tokens=*" %%b in (${bloblist_prefix}-bloblist-updated.txt) do ("C:/Program Files (x86)/Microsoft SDKs/Azure/CLI2/wbin/az.cmd" storage blob copy start --auth-mode login --account-name ${azure_storage_account} --source-container ${bucket} --source-blob "${environment}/latest/%%b" --destination-container ${bucket} --destination-blob "${environment}/%%b")
                    """
                    bat """"C:/Program Files (x86)/Microsoft SDKs/Azure/CLI2/wbin/az.cmd" storage blob delete-batch --auth-mode login --account-name ${azure_storage_account} -s ${bucket} --pattern ${environment}/latest/*.msi"""
                    bat """"C:/Program Files (x86)/Microsoft SDKs/Azure/CLI2/wbin/az.cmd" storage blob upload --auth-mode login --account-name ${azure_storage_account} -f artifacts/${foundationdi_installer_filename} -c ${bucket} -n ${environment}/latest/${foundationdi_installer_filename} """
                }
            }

        stage("Notifications", true){
            slackSend channel: 'aifoundation', color: 'good', message: "Completed ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", tokenCredentialId: 'slack-token'
        }
    } catch (Exception e) {
        echo "Error during build ${BUILD_NUMBER}: ${e}"
        slackSend channel: 'aifoundation', color: 'danger', message: "Failed build ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", tokenCredentialId: 'slack-token'
        throw e
    }
}