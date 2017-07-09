#!/usr/bin/groovy

// Global Variables
properties([
    [$class: 'ParametersDefinitionProperty', parameterDefinitions: [
        [$class: 'StringParameterDefinition', name: 'STI_IMAGE_NAME', defaultValue: 'php-56-rhel7:5.6-14', description: "STI Builder image name"]    ]]
])

node {
    echo "Build Number is: ${env.BUILD_NUMBER}"
    echo "Job Name is: ${env.JOB_NAME}"
    def commit_id, source, origin_url, branch, name
    stage ('Initialise') {
        // Checkout code from repository - we want commit id, name and branch
        checkout scm
        dir ("${WORKSPACE}") {
            commit_id = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim() 
            echo "Git Commit is: ${commit_id}"
            def cmd0 = $/name=$(git config --local remote.origin.url); name=$${name##*/}; echo $${name%%.git}/$
            name = sh(returnStdout: true, script: cmd0).trim()
            def cmd1 = $/git rev-parse --abbrev-ref HEAD/$            
            branch = sh(returnStdout: true, script: cmd1).trim()
            branch = branch.toLowerCase()
            name = "${name}-${branch}"
            echo "Name is: ${name}"
        }
        origin_url = sh(returnStdout: true, script: 'git config --get remote.origin.url').trim()
        source = "${origin_url}#${commit_id}"    
        echo "Source URL is: ${source}"
    }

    stage ('Scan') {
        // Scan code and CVE check
        updateCodeDeps()
        if (containsCVE()) {
            echo "CVE's found"
            timeout(time: 2, unit: 'DAYS') {
                def userInput = input(
                    id: 'userInput', message: 'CVE\'s found, Continue ?', parameters: [
                        [$class: 'BooleanParameterDefinition', defaultValue: false, description: 'Continue on CVE', name: 'Continue on CVE']]) 
            }
        }
        if (performance == false) {
            currentBuild.result = 'FAILURE'
        }
    }

    stage ('Build') {
        // Start Build or Create initial app if doesn't exist
        if(getBuildName(name)) {
            echo 'Building image'
            def build = getBuildName(name)
            setBuildRef(build, origin_url, commit_id)
            setBuildImage(build, "${STI_IMAGE_NAME}")
            openshiftBuild(buildConfig: build, showBuildLogs: 'true')
        } else {
            echo 'Creating app'
            try {
                sh "oc new-app ${STI_IMAGE_NAME}~${source} --name=${name} --labels=app=${name} --strategy=source || echo 'app exists'"
            } catch(Exception e) {
                echo "new-app exists"
            }
        }
    }

    stage ('Deploy') {
        echo 'Deploying image'
        def deploy = getDeployName(name)
        openshiftDeploy(deploymentConfig: deploy)
    }

    stage ('Create Route') {
        echo 'Creating a route to application'
        createRoute(name)
    }
}

// Expose service to create a route
def createRoute(String name) {
    try {
        def service = getServiceName(name)
        sh "oc expose svc ${service}"
    } catch(Exception e) {
        echo "route exists"
    }    
}

// Get Build Name
def getBuildName(String name) {
    def cmd2 = $/buildconfig=$(oc get bc -l app=${name} -o name);echo $${buildconfig##buildconfig*/}/$
    bld = sh(returnStdout: true, script: cmd2).trim()    
    return bld
}

// Get Deploy Config Name
def getDeployName(String name) {
    def cmd3 = $/deploymentconfig=$(oc get dc -l app=${name} -o name);echo $${deploymentconfig##deploymentconfig*/}/$
    dply = sh(returnStdout: true, script: cmd3).trim()    
    return dply
}

// Get Service Name
def getServiceName(String name) {
    def cmd4 = $/service=$(oc get svc -l app=${name} -o name);echo $${service##service*/}/$
    svc = sh(returnStdout: true, script: cmd4).trim()        
    return svc
}

// Set Build Ref
def setBuildRef(String build, String source, String commit_id) {
    def cmd5 = $/oc patch bc/"${build}" -p $'{\"spec\":{\"source\":{\"git\":{\"uri\":\"${source}\",\"ref\": \"${commit_id}\"}}}}$'/$
    sh cmd5
}

// Set Build Image
def setBuildImage(String build, String image) {
    def cmd6 = $/oc patch bc/"${build}" -p $'{\"spec\":{\"strategy\":{\"sourceStrategy\":{\"from\":{\"name\": \"${image}\"}}}}}$'/$
    sh cmd6
}

// Update code deps
def updateCodeDeps() {
    def cmd7 = $/php tools/composer.phar update/$
    sh cmd7
}

// Scan code deps for CVE's
def containsCVE() {
    def cmd7 = $/php tools/security-checker.phar security:check ./composer.lock/$
    check = sh(returnStdout: true, script: cmd7).trim()
    if (check.grep(~/CVE/)) {
        return true;
    }
    return false;
}
