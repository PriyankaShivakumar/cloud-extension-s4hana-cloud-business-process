#!/usr/bin/env groovy
@Library(['piper-lib', 'piper-lib-os', 'lmit-jenkins-lib']) _

pipeline {
    agent { label 'master' }
    parameters{
        string(name: 'credentialsId', defaultValue: 'pusercf',
            description: 'The user with which we can access BTP Landscape and CF')

        string(name: 'btpGlobalAccountId', defaultValue: '93951304-9109-44bc-ac3f-53c3ac8b309b',
            description: 'BTP Global AccountId')

        string(name: 'btpSubdomainName', defaultValue: 'GeoAuto',
            description: 'BTP subdomain name of the subaccount')

        string(name: 'displayNameInBTP', defaultValue: 'GeoRelAutomation',
            description: 'BTP subaccount display name')

        string(name: 'btpRegion', defaultValue: 'eu10',
            description: 'BTP region of the subaccount')
    
        string(name: 'btpRegion_cf_env', defaultValue: 'eu10-004',
            description: 'BTP region of the cloud foundry environment')
    
        string(name: 'btpLandscape', defaultValue: 'factory',
            description: 'BTP region of the subaccount')

        string(name: 'apiEndpoint', defaultValue: 'https://api.cf.eu10-004.hana.ondemand.com',
            description: 'CF api endpoint')
    
        string(name: 'cfOrgName', defaultValue: 'geoOrg',
            description: 'CF Org Name')

        string(name: 'cfSpaceName', defaultValue: 'dev',
            description: 'CF Space Name')
        
        string(name: 'hanaCloudTenantURL', defaultValue: 'https://my304263.s4hana.ondemand.com/ui#Shell-home',
            description: 'Hana Cloud Tenant URL')
        
        string(name: 'hanaCloudTenantusername', defaultValue: 'priyanka.r.s@sap.com',
            description: 'Hana Cloud Tenant Username')
        
        string(name: 'hanaCloudTenantPassword', defaultValue: 'Welcome@20',
            description: 'Hana Cloud Tenant Password')

        string(name: 'cockpitURL', defaultValue: 'https://cockpit.eu10.hana.ondemand.com/cockpit/#/globalaccount/93951304-9109-44bc-ac3f-53c3ac8b309b/accountModel',
            description: 'Cockpit URL')
        
        // Stages enabled by default
        booleanParam(name: 'createSubaccount', defaultValue: true,
            description: 'Create Subaccount, assign permissions and entitlements')
        booleanParam(name: 'deployApplication', defaultValue: true,
            description: 'Deploy the applications, Create destinations and assign permissions in BTP')
        booleanParam(name: 'runTests', defaultValue: true,
            description: 'Execute the Mocha tests to test the application')
        booleanParam(name: 'deleteSubaccount', defaultValue: true,
            description: 'Delete Subaccount in BTP')
    }
    options {
        buildDiscarder(
            logRotator(artifactDaysToKeepStr: '5', artifactNumToKeepStr: '10', daysToKeepStr: '15', numToKeepStr: '20')
        )
    }
    stages {
        stage('Create Subaccount, assign permissions and entitlements') {
            when {
                expression { params.createSubaccount == true }
            }
            steps {
                script {
                    subaccountStatus = checkSubaccount(
                        btpCredentialsId: params.credentialsId,
                        btpGlobalAccountId: params.btpGlobalAccountId,
                        btpSubdomainName: params.btpSubdomainName,
                        btpRegion: params.btpRegion,
                        btpLandscape: params.btpLandscape,
                    )
                    if (subaccountStatus == true){
                        try {
                            retry (5) {
                                cloudFoundryDeleteSpace(
                                    cfApiEndpoint: params.apiEndpoint,
                                    cfOrg: params.cfOrgName,
                                    cfSpace: params.cfSpaceName,
                                    cfCredentialsId: params.credentialsId,
                                    script: this
                                )
                            }
                            deleteSubaccount(
                                btpCredentialsId: params.credentialsId,
                                btpGlobalAccountId: params.btpGlobalAccountId,
                                btpSubdomainName: params.btpSubdomainName, 
                                btpRegion: params.cfEnvRegion,
                                btpLandscape: params.btpLandscape
                            )
                        } catch(e){
                            echo 'Cleanup failed before Subaccount Creation'
                        }    
                    }
                    createSubaccount(
                        btpCredentialsId: params.credentialsId,
                        btpGlobalAccountId: params.btpGlobalAccountId,
                        btpSubdomainName: params.btpSubdomainName,
                        displayNameInBTP: params.displayNameInBTP,
                        btpRegion: params.btpRegion,
                        btpLandscape: params.btpLandscape,
                        description: 'Geo Subaccount Creation'
                    )
                    createCFEnvironment(
                        script: this,
                        btpCredentialsId: params.credentialsId,
                        btpGlobalAccountId: params.btpGlobalAccountId,
                        btpSubdomainName: params.btpSubdomainName,
                        btpRegion: params.btpRegion,
                        btpRegionForCFEnv: params.btpRegion_cf_env,
                        btpLandscape: params.btpLandscape,
                        cfOrgName: params.cfOrgName
                    )
                    cloudFoundryCreateSpace(
                        cfApiEndpoint: params.apiEndpoint,
                        cfOrg: params.cfOrgName,
                        cfSpace: params.cfSpaceName,
                        cfCredentialsId: params.credentialsId,
                        script: this
                    )
                    assignRoleToUser(
                        btpCredentialsId: params.credentialsId,
                        btpGlobalAccountId: params.btpGlobalAccountId,
                        btpSubdomainName: params.btpSubdomainName,
                        btpRegion: params.btpRegion,
                        btpLandscape: params.btpLandscape,
                        roleCollection: 'Subaccount Administrator'
                    )
                    assignEntitlements(
                        btpCredentialsId: params.credentialsId,
                        btpGlobalAccountId: params.btpGlobalAccountId,
                        btpSubdomainName: params.btpSubdomainName,
                        entitlements: [
                            ['serviceName': 'enterprise-messaging', 'plan': 'default'],
                            ['serviceName': 'APPLICATION_RUNTIME', 'plan': 'MEMORY', 'quota': '1']
                        ]
                    )

                }
            }
        }   
        stage('Register Hana Cloud System, Create service instances and Deploy the applications') {
            when {
                expression { params.deployApplication == true }
            }
            steps {
                script {
                    dockerExecuteOnKubernetes(script: this, dockerImage: 'docker.wdf.sap.corp:51010/sfext:v3' ){
                        withCredentials([usernamePassword(credentialsId: params.credentialsId, passwordVariable: 'password', usernameVariable: 'username')]) {
               
                            build job: 'Georel_RegisterSystem', parameters: [[$class: 'StringParameterValue', name: 'URL', value: cockpitURL],[$class: 'StringParameterValue', name: 'Username', value: username],[$class: 'StringParameterValue', name: 'Password', value: password],[$class: 'StringParameterValue', name: 'hanaCloudTenantURL', value: hanaCloudTenantURL],[$class: 'StringParameterValue', name: 'hanaCloudTenantusername', value: hanaCloudTenantusername],[$class: 'StringParameterValue', name: 'hanaCloudTenantPassword', value: hanaCloudTenantPassword],[$class: 'StringParameterValue', name: 'subAccountName', value: displayNameInBTP]]
                            
                            git branch: 'us10',credentialsId: 'I559299_wdf',url: 'https://github.wdf.sap.corp/CentralAutomationTeam/CloudPortal.git'
			    properties = readProperties file: 'config.properties'
		            systemName= properties['SystemName']
		            print systemName

                            topicid = (Math.abs(new Random().nextInt(9000))+1000).toString()
                            print topicid

                            checkout scm
                            
                            sh '''
                                npm config set unsafe-perm true
			        npm rm -g @sap/cds
			        npm i -g @sap/cds-dk
			        cd georel
                                npm install
                                npm install sqlite3
                                cds deploy --to sqlite
			        cd ..
                            '''
                            
                            georelmessagingJson = readJSON file: 'georelmessaging.json'
			    georelmessagingJson.emClientId = topicid
                            georelmessagingJson.systemName = systemName
                            writeJSON file: 'georelmessaging.json', json: georelmessagingJson
                            sh "cat georelmessaging.json"

                            apiaccessJson = readJSON file: 'apiaccess.json'
			    communication_arr = "Comm_Arr_" + "${topicid}"
                            apiaccessJson.communicationArrangement.communicationArrangementName = communication_arr
                            apiaccessJson.systemName = systemName
                            writeJSON file: 'apiaccess.json', json: apiaccessJson
                            sh "cat apiaccess.json"

                            eventmeshJson = readJSON file: 'eventmesh.json'
                            topicnamespace = "sap/S4HANAOD/" + "${topicid}" + "/*"
			    eventmeshJson.rules.queueRules.subscribeFilter[1] = topicnamespace
                            eventmeshJson.rules.topicRules.subscribeFilter[1] = topicnamespace
                            writeJSON file: 'eventmesh.json', json: eventmeshJson
                            sh "cat eventmesh.json"

                            cloudFoundryCreateService(
                                cfApiEndpoint: params.apiEndpoint,
                                cfOrg: params.cfOrgName,
                                cfSpace: params.cfSpaceName,
                                cfCredentialsId: params.credentialsId,
                                cfService: 's4-hana-cloud',
                                cfServiceInstanceName: 'georelmessaging',
                                cfServicePlan: 'messaging',
                                cfCreateServiceConfig: 'georelmessaging.json',
                                script: this
                            )

                            cloudFoundryCreateService(
                                cfApiEndpoint: params.apiEndpoint,
                                cfOrg: params.cfOrgName,
                                cfSpace: params.cfSpaceName,
                                cfCredentialsId: params.credentialsId,
                                cfService: 's4-hana-cloud',
                                cfServiceInstanceName: 'xf_api_bupa',
                                cfServicePlan: 'api-access',
                                cfCreateServiceConfig: 'apiaccess.json',
                                script: this
                            )

                            cloudFoundryCreateService(
                                cfApiEndpoint: params.apiEndpoint,
                                cfOrg: params.cfOrgName,
                                cfSpace: params.cfSpaceName,
                                cfCredentialsId: params.credentialsId,
                                cfService: 'enterprise-messaging',
                                cfServiceInstanceName: 'messaging_georel',
                                cfServicePlan: 'default',
                                cfCreateServiceConfig: 'eventmesh.json',
                                script: this
                            )
                            getServiceCreateStatus(
                                btpCredentialsId: params.credentialsId,
                                orgName: params.cfOrgName,
                                btpRegion: params.btpRegion,
                                btpRegionForCFEnv: params.btpRegion_cf_env,
                                spaceName: params.cfSpaceName,
                                serviceInstanceName: 'messaging_georel'
                            )
                            getServiceCreateStatus(
                                btpCredentialsId: params.credentialsId,
                                orgName: params.cfOrgName,
                                btpRegion: params.btpRegion,
                                btpRegionForCFEnv: params.btpRegion_cf_env,
                                spaceName: params.cfSpaceName,
                                serviceInstanceName: 'xf_api_bupa'
                            )
                            getServiceCreateStatus(
                                btpCredentialsId: params.credentialsId,
                                orgName: params.cfOrgName,
                                btpRegion: params.btpRegion,
                                btpRegionForCFEnv: params.btpRegion_cf_env,
                                spaceName: params.cfSpaceName,
                                serviceInstanceName: 'georelmessaging'
                            )

                            build job: 'Georel_AddOutboundTopic', parameters: [[$class: 'StringParameterValue', name: 'URL', value: cockpitURL],[$class: 'StringParameterValue', name: 'Username', value: username],[$class: 'StringParameterValue', name: 'Password', value: password],[$class: 'StringParameterValue', name: 'hanaCloudTenantURL', value: hanaCloudTenantURL],[$class: 'StringParameterValue', name: 'hanaCloudTenantusername', value: hanaCloudTenantusername],[$class: 'StringParameterValue', name: 'hanaCloudTenantPassword', value: hanaCloudTenantPassword],[$class: 'StringParameterValue', name: 'topicid', value: topicid]]
                            
			    topicJson = readJSON file: 'georel/srv/topic.json'
			    ns = "sap/S4HANAOD/" + "${topicid}"
                            topicJson.namespace = ns
                            writeJSON file: 'georel/srv/topic.json', json: topicJson
                            sh "cat georel/srv/topic.json"
        
                            cloudFoundryDeploy(
                                script: this,
                                deployTool: 'cf_native',
                                cfCredentialsId: params.credentialsId,
                                cfApiEndpoint: params.apiEndpoint,
                                cfOrg: params.cfOrgName,
                                cfSpace: params.cfSpaceName,
                                cloudFoundry: [manifest: 'manifest.yaml']
                            )
                        }
                    }
                }
            }
        }
        stage('Demoscript') {
            when {
                expression { params.runTests == true }
            }
            steps {
                script {
                    dockerExecuteOnKubernetes(script: this, dockerEnvVars: ['pusername':pusername, 'puserpwd':puserpwd], dockerImage: 'docker.wdf.sap.corp:51010/sfext:v3' ){
                        withCredentials([usernamePassword(credentialsId: params.credentialsId, passwordVariable: 'password', usernameVariable: 'username')]) {
                            sh "cf login -a $apiEndpoint -u $username -p $password -o $cfOrgName -s $cfSpaceName"  
                            sh '''
			        curl -L https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64 --output jq
                                chmod +x jq
                                mv jq /usr/local/bin/jq
				
                                appId=`cf app geoe2e --guid`
                                `cf curl /v2/apps/$appId/env > ab.txt`
                            '''
                            def app_url = sh([ 
                                script: """cat ab.txt | jq -r '.application_env_json.VCAP_APPLICATION.application_uris[0]'""", 
                                returnStdout: true 
                            ]).trim()

                            def application_url= "https://${app_url}/georelApplication.html"
                            sh([
                                script: """echo '${application_url}'""",
                                returnStdout: true
                            ])
                             
                            build job: 'Georel_Demoscript', parameters: [[$class: 'StringParameterValue', name: 'URL', value: cockpitURL],[$class: 'StringParameterValue', name: 'Username', value: username],[$class: 'StringParameterValue', name: 'Password', value: password],[$class: 'StringParameterValue', name: 'hanaCloudTenantURL', value: hanaCloudTenantURL],[$class: 'StringParameterValue', name: 'hanaCloudTenantusername', value: hanaCloudTenantusername],[$class: 'StringParameterValue', name: 'hanaCloudTenantPassword', value: hanaCloudTenantPassword],[$class: 'StringParameterValue', name: 'topicid', value: topicid],[$class: 'StringParameterValue', name: 'SystemName', value: systemName],[$class: 'StringParameterValue', name: 'AppURL', value: application_url]]

                        }    
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                if (params.deleteSubaccount == true) {
		    try {
                        retry (5) {
                            cloudFoundryDeleteSpace(
                                cfApiEndpoint: params.apiEndpoint,
                                cfOrg: params.cfOrgName,
                                cfSpace: params.cfSpaceName,
                                cfCredentialsId: params.credentialsId,
                                script: this
                            )
                        }
                        deleteSubaccount(
                            btpCredentialsId: params.credentialsId,
                            btpGlobalAccountId: params.btpGlobalAccountId,
                            btpSubdomainName: params.btpSubdomainName, 
                            btpRegion: params.cfEnvRegion,
                            btpLandscape: params.btpLandscape
                        )
                        afterDeleteStatus = checkSubaccount(
                            btpCredentialsId: params.credentialsId,
                            btpGlobalAccountId: params.btpGlobalAccountId,
                            btpSubdomainName: params.btpSubdomainName, 
                            btpRegion: params.cfEnvRegion,
                            btpLandscape: params.btpLandscape
                        )
                        assert afterDeleteStatus == false
			    
			build job: 'Georel_RemoveSystem', parameters: [[$class: 'StringParameterValue', name: 'URL', value: cockpitURL],[$class: 'StringParameterValue', name: 'Username', value: username],[$class: 'StringParameterValue', name: 'Password', value: password],[$class: 'StringParameterValue', name: 'SystemName', value: systemName]]

                    } catch(e){
                        echo 'Cleanup failed'
                    }
                }
                cleanWs()
                emailext body: '$DEFAULT_CONTENT', subject: '$DEFAULT_SUBJECT', to: 'priyanka.r.s@sap.com'   
            }
        }
    }
}
