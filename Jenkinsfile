#!groovy
import groovy.json.JsonSlurperClassic
node {

    // Test Scratch Org Username
    def SFDC_USERNAME

    // Path to SF CLI (configured via Custom Tools)
    def toolbelt = tool 'toolbelt'

    echo sh(script: 'env|sort', returnStdout: true)

    stage('Checkout Source') {
        checkout scm
    }

    withCredentials([file(credentialsId: env.JWT_CRED_ID_DH, variable: 'jwt_key_file')]) {

        stage('Create Test Scratch Org') {

            // Authorizate with DevHub via JWT grant
            rc = sh returnStatus: true, script: "${toolbelt}/sf org login jwt --client-id ${env.CONNECTED_APP_CONSUMER_KEY_DH} --username ${env.HUB_ORG_DH} --jwt-key-file ${jwt_key_file} --instance-url ${env.SFDC_HOST_DH}"
            if (rc != 0) { error 'hub org authorization failed' }

            // Create Scratch Org and determine login username X
            rmsg = sh returnStdout: true, script: "${toolbelt}/sf org create scratch --target-dev-hub ${env.HUB_ORG_DH} --definition-file config/project-scratch-def.json --json"
            def robj = new JsonSlurperClassic().parseText(rmsg)
            if (robj.status != 0) { error 'org creation failed: ' + robj.message }
            SFDC_USERNAME=robj.result.username
        }

        stage('Push To Test Scratch Org') {

            // Push code via sf force:source:push
            rc = sh returnStatus: true, script: "${toolbelt}/sf project deploy start --target-org ${SFDC_USERNAME}"
            if (rc != 0) {
                error 'push failed'
            }
        }

        stage('Run Tests') {

            // Create test output directory, run tests 
            sh "mkdir -p tests/${env.BUILD_NUMBER}"

            // Run Apex Tests
            rc = sh returnStatus: true, script: "${toolbelt}/sf apex run test --test-level RunLocalTests --output-dir tests/${env.BUILD_NUMBER} --result-format junit --target-org ${SFDC_USERNAME}"
            
            // Run Lightning Web Component Tests
            env.NODEJS_HOME = "${tool 'node'}"
            env.PATH="${env.NODEJS_HOME}/bin:${env.PATH}"            
            sh 'npm install'                
            rc = sh returnStatus: true, script: 'npm run test:unit'                
            
            // Have Jenkins capture the test results
            junit keepLongStdio: true, testResults: 'tests/**/*-junit.xml'
            junit keepLongStdio: true, testResults: 'junit.xml'
        }

        stage('Delete Test Org') {

            // Delete Test Scratch Org 
            rc = sh returnStatus: true, script: "${toolbelt}/sf org delete scratch --target-org ${SFDC_USERNAME}"
            if (rc != 0) {
                error 'org delete failed'
            }            
        }
    }
}