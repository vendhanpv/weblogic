#!groovy
library 'reference-pipeline'
library 'AppServiceAccount'
library 'CICD-FOSS-V2'
library 'lhl_cc_library'

properties([
		pipelineTriggers([[$class:"com.cloudbees.jenkins.GitHubPushTrigger"]])
])

pipeline {
	
        parameters  {
              string(name: 'BRANCH_NAME', defaultValue: '', description: 'branch to build')
	}
	agent any

	options {
		buildDiscarder(logRotator(numToKeepStr: '20'))
	}
        parameters    {
               string(name: 'DEPLOY_VERSION', defaultValue: '', description: 'Version to push to production, pulled from Nexus')
               string(name: 'BRANCH_NAME', defaultValue: '', description: 'The branch to build' )
        }
	tools {
		nodejs 'NodeJS-10.14.1'
		jdk 'JAVA_8'
	}

	environment {

		//General Application Information
		APP_NAME="fxg-lhl-execute-trip-mobile"
		NEXUS_APP_NAME="FXG-LHL-ExecuteTrip-Mobile-3536859"
		APP_GROUP="eai3536859.com.fedex.ground.transportation.trip"


		APP_PAM_ID = '1696001' //get from Systems team
		CF_ORG="3536859"
		CF_API_DEV_RELEASE = "https://api.sys.clwdev1.paas.fedex.com"

		NEXUS_LOGIN = credentials('FXS_git_credentials_3536859')
		NEXUS_URL_DOWNLOAD = 'https://nexus.prod.cloud.fedex.com:8443/nexus/service/local/artifact/maven/content'
		NEXUS_URL_UPLOAD = 'nexus.prod.cloud.fedex.com:8443/nexus'
		SNAPSHOT_REPO_NAME = 'snapshot'
		RELEASE_REPO_NAME = 'release'

		//Notifications
		TEAMS_WEBHOOK = 'https://outlook.office.com/webhook/ec696683-91ed-4190-a9b3-08b10d2bba32@b945c813-dce6-41f8-8457-5a12c2fe15bf/IncomingWebhook/fe8e8868ad6542c682f28c4f4193c166/24e06ece-f181-426c-979f-bf2cc6f61793'

		//GIT_COMMIT_HASH = sh (script: "git rev-parse --short HEAD", returnStdout: true).trim()
		clwdev1='{"CF_SPACE":"development","MANIFEST_FILE_NAME":"manifest-development-clwdev1.yml","CF_API":"https://api.sys.clwdev1.paas.fedex.com","CONFIG_SERVER_URL":"https://fxg-lhl-pltfmsvcs-config-service-development.app.pghdev1.paas.fedex.com","SERVICE_REGISTRY_URL":"https://fxg-lhl-pltfsvc-svc-registry-1-development.app.pghdev1.paas.fedex.com/eureka,https://fxg-lhl-pltfsvc-svc-registry-2-development.app.pghdev1.paas.fedex.com/eureka"}'
		clwdev2='{"CF_SPACE":"development","MANIFEST_FILE_NAME":"manifest-development-clwdev2.yml","CF_API":"https://api.sys.clwdev2.paas.fedex.com","CONFIG_SERVER_URL":"https://fxg-lhl-pltfmsvcs-config-service-development.app.pghdev1.paas.fedex.com","SERVICE_REGISTRY_URL":"https://fxg-lhl-pltfsvc-svc-registry-1-development.app.pghdev1.paas.fedex.com/eureka,https://fxg-lhl-pltfsvc-svc-registry-2-development.app.pghdev1.paas.fedex.com/eureka"}'
		clwdev3='{"CF_SPACE":"development","MANIFEST_FILE_NAME":"manifest-development-clwdev3.yml","CF_API":"https://api.sys.clwdev3.paas.fedex.com","CONFIG_SERVER_URL":"https://fxg-lhl-pltfmsvcs-config-service-development.app.pghdev1.paas.fedex.com","SERVICE_REGISTRY_URL":"https://fxg-lhl-pltfsvc-svc-registry-1-development.app.pghdev1.paas.fedex.com/eureka,https://fxg-lhl-pltfsvc-svc-registry-2-development.app.pghdev1.paas.fedex.com/eureka"}'

		development="clwdev1,clwdev2,clwdev3"
	}

	stages {

		stage("Checkout") {
			steps {
				cleanWs()
				echo 'Stage checkout'
				script {
				            if("${env.gitlabTargetBranch} = null") {
                   						echo "checking out $params.BRANCH_NAME"
                   						SOURCE_BRANCH = params.BRANCH_NAME

                   			}else {
                   					echo "GitlabTargetBranch: "+gitlabTargetBranch
                   					SOURCE_BRANCH = "${env.gitlabTargetBranch}"
                   				 }
				      }
				git url: "git@github.com:FedEx/eai-3536859-fxg-lhl-execute-trip-mobile.git", branch: "${SOURCE_BRANCH}", credentialsId: '3536859-fxg-lhl-execute-trip-mobile-deploy-key'
				stash allowEmpty: true,includes: "manifests/*", name: "manifestFiles"
			}
		}

		stage("Build") {
		  environment{
                gateway_base_url = credentials('api_gateway_url_test');
          }
		agent {
			docker {
				label 'docker'
				image 'nexus2.prod.cloud.fedex.com:8444/fdx/jenkins/default-tools-image'
			}
		}
			steps {
				script {
					sh 'npm cache --force clean'
        			sh 'npm config set registry	https://nexus.prod.cloud.fedex.com:8443/nexus/content/groups/registry-npm-fedex/'
					sh 'chmod +x gradlew'
          
						GIT_COMMIT_HASH = sh (script: "git rev-parse --short HEAD", returnStdout: true).trim()
						echo 'after git rev-parse'
						VERSION = sh (script: "./gradlew -P revision=$GIT_COMMIT_HASH properties -q | grep \"version:\" | awk '{print \$2}'", returnStdout: true).trim()
						echo "print version on line.."
						echo "$VERSION"
				}
					sh """#!/bin/bash
					 export gateway_base_url="$gateway_base_url"
					./gradlew -P revision=$GIT_COMMIT_HASH clean build """
			}
			post {
                
				success {
					echo 'Build complete'
					stash includes: '**/', excludes: "**/src/main/webapp/node_modules/**", name: 'buildDir'
					stash includes: "build/libs/${APP_NAME}-${version}" + ".jar", name: "builtArtifact"
					stash includes: '**/src/main/webapp/node_modules/**', name: 'node_modules'
					
				}
			}
		}
		
		stage('DAST') {
        	steps {
        	  qualysWASScan (
        	   authRecord: 'none',
        	   cancelOptions: 'none',
        	   credsId: 'Qualys-CICD',
        	   optionProfile: 'useDefault',
        	   platform: 'US_PLATFORM_1',
        	   pollingInterval: '5',
        	   proxyPassword: '',
        	   proxyPort: 3128,
        	   proxyServer: 'internet.proxy.fedex.com',
        	   proxyUsername: '',
        	   scanName: '[job_name]_jenkins_build_[build_number]',
        	   scanType: 'VULNERABILITY',
        	   useProxy: true,
        	   vulnsTimeout: '60*24',
        	   webAppId: '413227662'
        	   )
        	}
        }
           
		stage('Unit Tests') {
			agent {
				docker {
					label 'docker'
					image 'nexus2.prod.cloud.fedex.com:8444/fdx/jenkins/headless-chrome-image'
				}
			}
			 when {
                expression {
                    (SOURCE_BRANCH != 'Integration')
                }
             }
			steps {
				sh """
					wget https://nexus.prod.cloud.fedex.com:8443/nexus/repository/devframeworkrepo/com/fedex/devopsys/external/njswrapper/external-njswrapper-1/01/01-eirslett-lib/external-njswrapper-1.01.01-eirslett-lib-1.6/1.6/external-njswrapper-1.01.01-eirslett-lib-1.6-1.6.tar
					tar xf external-njswrapper-1.01.01-eirslett-lib-1.6-1.6.tar

					cd src/main/webapp

					# Needed for unit testing
					sed  -i  '1i const puppeteer = require("puppeteer");' ${WORKSPACE}/src/main/webapp/src/karma.conf.js
					sed  -i  '2i process.env.CHROME_BIN = puppeteer.executablePath();' ${WORKSPACE}/src/main/webapp/src/karma.conf.js
					sed  -i  '3i const browser = puppeteer.launch({args: ["--no-sandbox"]});' ${WORKSPACE}/src/main/webapp/src/karma.conf.js

					npm install
					npm install karma-junit-reporter --save-dev

					chmod 755 -R node_modules
						export CHROME_BIN=${WORKSPACE}/node_modules/puppeteer/.local-chromium/linux-499413/chrome-linux/chrome
						export LD_LIBRARY_PATH=${WORKSPACE}/lib/chrome/linux-x64

					npm test

					"""
			}
			post {
				success {
					echo 'Unit Tests Success'
						stash includes: '**/coverage/*', name: 'testCoverage'
				}
			}
		}
		
       
           
		stage("SonarQube reports upload") {
		  agent {
            docker {
                label 'docker'
                image 'nexus2.prod.cloud.fedex.com:8444/fdx/jenkins/default-tools-image'
            }
          }
           when {
              expression {
                  (SOURCE_BRANCH != 'Integration')
              }
           }
			environment {
				VERSION = "$version"
			}
			steps {
				unstash 'buildDir'
				unstash 'testCoverage'
				withSonarQubeEnv('SonarQube') {
					sh "ls -ltr"
					sh "find src/main/webapp/coverage -name '*.*'"
					sh 'chmod +x gradlew'
					sh "./gradlew appNpmSonar -P revision=$GIT_COMMIT_HASH"
				}
			}
		}

		/*stage("Quality Gate") {
		    when {
                expression {
                    (SOURCE_BRANCH != 'Integration')
                }
            }
			steps {
				sonarQualityGate()
			}
		}*/

		stage('SonarQube Permissions') {
		    when {
                expression {
                    (SOURCE_BRANCH != 'Integration')
                }
            }
			steps {
				sonarQubePermissions projectkey: "$APP_NAME", eai: '3536859'
			}
		}
		stage("stashing source ") {
            steps {
                stash includes: '**', name: 'source'
            }
        } 

		stage('Get fortify scripts') {
            steps {
                getFortifyScripts()
            }
        }

        stage('Run Fortify Analysis') {
            steps {
                echo 'Started unit testing'
                startFortifyAnalysis("${CF_ORG}_${APP_NAME}")
            }
        }

		stage('Environment Setup') {
            steps {
                script {
                    def release = readYaml file: 'release.yml'
                    APP_VERSION = release.releaseVersion
                    "Version retrieved from release.yml: ${APP_VERSION}"
                }
            }
        }    
		stage('FOSS'){

          steps{
			  unstash 'builtArtifact'
              script {
                  withCredentials([usernamePassword(credentialsId: 'Gitlab_3536859', passwordVariable: 'password', usernameVariable: 'username')]) { 
                        def nexusEval = runNexusPolicyEvaluation iqApplication: "3536859-fxg-lhl-execute-trip-mobile", 
                            iqStage: "build", 
                            iqTarget: "build/libs/",
                            svcUser: username,
                            svcPwd: password
                        saveNexusPolicyEvaluation iqApplication: "3536859-fxg-lhl-execute-trip-mobile",
                            applicationVersion: "$APP_VERSION",
                            iqRptId: nexusEval.get("iqRptId"),
                            zone: 'Internal Usage - Back Office (Includes COPE mobile Devices, Desktop Development, Testing, support and maintenance)',
                            svcUser: username,
                            svcPwd: password
                        submitFossRequestForNexusPolicyEvaluation iqApplication: "3536859-fxg-lhl-execute-trip-mobile",
                            applicationVersion: "$APP_VERSION",
                            svcUser: username,
                            svcPwd: password 
							failOnError: false;
    	            }
              }
          } 
        }      
        
		stage('RelyBP Static') {
                  steps {
                      sonarqube (
                          projectKey: "${CF_ORG}_${APP_NAME}",
                          projectName: "${APP_NAME}",
                          src: "src/main/webapp",
                          binaries: "build",
                          projectVersion: "1.0",
                          eai: "${CF_ORG}",
                          enableRelyBP: "true"
                      )
                  }
              }
		
		stage("Setup Services in development space clwdev1") {
			environment{
				CONFIG_SERVER_URL = "https://fxg-lhl-execute-trip-mobile-svc-config-service-development.app.clwdev1.paas.fedex.com/"
			}
			steps {
				println 'Configuring User Provided Services'

				pcfDeploy pamId: APP_PAM_ID,
						url: "https://api.sys.clwdev1.paas.fedex.com",
						space: "development",
						cfcmd: 'version'

				sh '''#!/bin/sh
                    chmod +x cf
                    export PATH=${PATH}:${WORKSPACE}
                    currentServices=$(cf services)
                    echo ${currentServices}

           
					if [[ "${currentServices}" != *"$APP_NAME-config-server"* ]]; then
						cf cups $APP_NAME-config-server -p '{\"apiUrl\":\"'${CONFIG_SERVER_URL}'\"}'
					fi
                '''
			}
		}
		stage("Deploy to development space clwdev1") {
            environment {
                VERSION = "$version"
                CF_API="https://api.sys.clwdev1.paas.fedex.com"
                CF_SPACE = "development"
            }
            steps {
                println 'running deploy-develop'
                unstash 'builtArtifact'
                simplePcfDeploy([
                        appName             : "$APP_NAME",
                        pamId               : APP_PAM_ID,
                        pcfApiUrl           : CF_API,
                        pcfOrg              : CF_ORG,
                        pcfSpace            : CF_SPACE,
                        jarFileLocation     : "build/libs/${APP_NAME}-${VERSION}.jar",
                        manifestFileLocation: "manifests/manifest-development-clwdev1.yml",
						services            : [
								[ service: 'appdynamics', plan: 'fedex1-test', serviceInstance: 'Shared AppD' ]
						],
                        environmentVariables: [[name: 'nexusVersion', value: '${VERSION}'],
                                               [name : 'SPRING_PROFILES_ACTIVE' ,value: "cloud"]]
                ]);
            }
        }
		stage("Setup Services in development space clwdev2") {
			environment{
				CONFIG_SERVER_URL = "https://fxg-lhl-execute-trip-mobile-svc-config-service-development.app.clwdev2.paas.fedex.com/"
			}
			steps {
				println 'Configuring User Provided Services'

				pcfDeploy pamId: APP_PAM_ID,
						url: "https://api.sys.clwdev2.paas.fedex.com",
						space: "development",
						cfcmd: 'version'

				sh '''#!/bin/sh
                    chmod +x cf
                    export PATH=${PATH}:${WORKSPACE}
                    currentServices=$(cf services)
                    echo ${currentServices}

           
					if [[ "${currentServices}" != *"$APP_NAME-config-server"* ]]; then
						cf cups $APP_NAME-config-server -p '{\"apiUrl\":\"'${CONFIG_SERVER_URL}'\"}'
					fi
                '''
			}
		}
		stage("Deploy to development space clwdev2") {
            environment {
                VERSION = "$version"
                CF_API="https://api.sys.clwdev2.paas.fedex.com"
                CF_SPACE = "development"
            }
            steps {
                println 'running deploy-develop'
                unstash 'builtArtifact'
                simplePcfDeploy([
                        appName             : "$APP_NAME",
                        pamId               : APP_PAM_ID,
                        pcfApiUrl           : CF_API,
                        pcfOrg              : CF_ORG,
                        pcfSpace            : CF_SPACE,
                        jarFileLocation     : "build/libs/${APP_NAME}-${VERSION}.jar",
                        manifestFileLocation: "manifests/manifest-development-clwdev2.yml",
						services            : [
								[ service: 'appdynamics', plan: 'fedex1-test', serviceInstance: 'Shared AppD' ]
						],
                        environmentVariables: [[name: 'nexusVersion', value: '${VERSION}'],
                                               [name : 'SPRING_PROFILES_ACTIVE' ,value: "cloud"]]
                ]);
            }
        }

		stage("Setup Services in development space clwdev3") {
			environment{
				CONFIG_SERVER_URL = "https://fxg-lhl-execute-trip-mobile-svc-config-service-development.app.clwdev3.paas.fedex.com/"
			}
			steps {
				println 'Configuring User Provided Services'

				pcfDeploy pamId: APP_PAM_ID,
						url: "https://api.sys.clwdev3.paas.fedex.com",
						space: "development",
						cfcmd: 'version'

				sh '''#!/bin/sh
                    chmod +x cf
                    export PATH=${PATH}:${WORKSPACE}
                    currentServices=$(cf services)
                    echo ${currentServices}

           
					if [[ "${currentServices}" != *"$APP_NAME-config-server"* ]]; then
						cf cups $APP_NAME-config-server -p '{\"apiUrl\":\"'${CONFIG_SERVER_URL}'\"}'
					fi
                '''
			}
		}
		stage("Deploy to development space clwdev3") {
            environment {
                VERSION = "$version"
                CF_API="https://api.sys.clwdev3.paas.fedex.com"
                CF_SPACE = "development"
            }
            steps {
                println 'running deploy-develop'
                unstash 'builtArtifact'
                simplePcfDeploy([
                        appName             : "$APP_NAME",
                        pamId               : APP_PAM_ID,
                        pcfApiUrl           : CF_API,
                        pcfOrg              : CF_ORG,
                        pcfSpace            : CF_SPACE,
                        jarFileLocation     : "build/libs/${APP_NAME}-${VERSION}.jar",
                        manifestFileLocation: "manifests/manifest-development-clwdev3.yml",
						services            : [
								[ service: 'appdynamics', plan: 'fedex1-test', serviceInstance: 'Shared AppD' ]
						],
                        environmentVariables: [[name: 'nexusVersion', value: '${VERSION}'],
                                               [name : 'SPRING_PROFILES_ACTIVE' ,value: "cloud"]]
                ]);
            }
        }

		stage ("RelyBP Development") {
            steps {
                pcfRelyDynamic (
                    pam_ID: APP_PAM_ID,
                    url: CF_API_DEV_RELEASE,
                    space: "development",
                    sonar_projectname: APP_NAME,
                    App_Name: APP_NAME,
                    username: "",
                    password: "",
                    envUrlPath: "/grdlhldispatch/actuator/env"
                )
            }
    	}
	}


	post {
		always{
			metricCollection(eai: "3536859", trainName: "LHL_RUNAWAY")
		}
		success {
			office365ConnectorSend color: '#00FF00', message: "#### Jenkins build successful for '${JOB_NAME}' \n please see output URL below\n  (<'${BUILD_URL}'|Open>) ", status: "Build Success", webhookUrl: TEAMS_WEBHOOK
		}
		failure {
			office365ConnectorSend color: '#FF0000', message: "#### Jenkins build failed for '${JOB_NAME}' \n please see output URL below\n  (<'${BUILD_URL}'|Open>) ", status: "Build Failure", webhookUrl: TEAMS_WEBHOOK
		}
	}
}
