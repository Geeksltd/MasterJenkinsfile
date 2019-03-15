def runPowershell(cmd,quiet = true) {
       def script = "powershell -ExecutionPolicy ByPass -command "+cmd;
       if(!quiet)
        script = "@echo off && " + script;

       return bat(returnStdout:true , script: script).trim()
}

import com.cloudbees.plugins.credentials.impl.*;
import com.cloudbees.plugins.credentials.*;
import com.cloudbees.plugins.credentials.domains.*;

pipeline 
{
    environment 
    {   
        REPOSITORY_CREDENTIALS_ID = "REPOSITORY_CREDENTIALS"	
        IMAGE_BUILD_VERSION_TAG = "v_${BUILD_NUMBER}"
		IMAGE_BUILD_VERSION = "${CONTIANER_REPOSITORY_URL}:${IMAGE_BUILD_VERSION_TAG}" 
		IMAGE_LATEST_VERSION = "${CONTIANER_REPOSITORY_URL}:latest" 
		ACCELERATE_PACKAGE_FILENAME="packages.json"
		PROJECT_REPOSITORY_PASSWORD="$PROJECT_REPOSITORY_PASSWORD"
		PROJECT_REPOSITORY_USERNAME="$PROJECT_REPOSITORY_USERNAME"
		PROJECT_REPOSITORY_URL = "$PROJECT_REPOSITORY_URL"
        PROJECT_REPOSITORY_BRANCH = "$BRANCH"
        AWS_REGION="$REGION"                
        TASK_ROLE_ARN ="${TASK_ROLE_NAME}-runtime" 
        ErrorActionPreference ="Stop"  
        DATABASE_NAME="$DATABASE_NAME"    
        S3_BACKUPS_BUCKET="$S3_BACKUPS_BUCKET"
        DEPLOYMENT_TAG_PREFIX="$DEPLOYMENT_TAG_PREFIX"
        PROJECT="$PROJECT"
        REFERENCE_DATABASE_BACKUP_NAME="$REFERENCE_DATABASE_BACKUP_NAME"
    }
    agent any
    stages
	{    
            stage('Init')
            {
                steps
                    {
                        script
                        {
                            ROLLBACK_DATABASE = false
                            HAS_DB_CHANGED = false                            
                            IS_APPLICATION_OFFLINE = false                          
                            S3_DATABASE_BACKUP_LOCATION="$S3_BACKUPS_BUCKET/$PROJECT/"
                            DATE_TAG=new Date().format('dd.MM.yyyy@hh.mm.ss');
                            REFERENCE_DATABASE_NAME = "$DATABASE_NAME" + "_" + DATE_TAG;
                            COMPLETION_TAG_NAME="$DEPLOYMENT_TAG_PREFIX" + DATE_TAG
                        }
                    }
            }
	         stage('Prepare credentials') 
            {
                steps
                {
                    script
                        {	
                            
							def repo_credentials = (Credentials) new UsernamePasswordCredentialsImpl(CredentialsScope.GLOBAL,REPOSITORY_CREDENTIALS_ID, "description", "$PROJECT_REPOSITORY_USERNAME", "$PROJECT_REPOSITORY_PASSWORD")
							SystemCredentialsProvider.getInstance().getStore().removeCredentials(Domain.global(), repo_credentials)
                            SystemCredentialsProvider.getInstance().getStore().addCredentials(Domain.global(), repo_credentials)
                        }
                }				
            }			
            stage('Clone source') 
            {
                steps
                {
                    script
                        {						
							git credentialsId: REPOSITORY_CREDENTIALS_ID,  url: PROJECT_REPOSITORY_URL, branch: PROJECT_REPOSITORY_BRANCH
                        }
                }				
            }
			
			stage('Update settings') 
            {
                steps
                {
                    script
                        {						
							bat 'replace-in-file -m'
                        }
                }				
            }
			
			stage('Build the source code') 
            {
                steps
                {
                    script
                        {	
                            bat "docker build -t $IMAGE_BUILD_VERSION -t $IMAGE_LATEST_VERSION ."
                        }
                }				
            }

			stage('Push the image') 
            {
                steps
                {
                    script
                        {
							powershell label: '', returnStatus: true, script: """Invoke-Expression -Command (Get-ECRLoginCommand -Region eu-west-1).Command 
                                                                                 docker push $IMAGE_BUILD_VERSION 
                                                                                 docker push $IMAGE_LATEST_VERSION"""
                        }
                }				
            }

            stage('Update database changes')
            {
                steps
                {
                    script
                        {   
                            
                           if(runPowershell("getDBChangeScripts")) 
                            {
                                HAS_DB_CHANGED = true;
                                runPowershell("applyDatabaseChanges $DATABASE_NAME $REFERENCE_DATABASE_NAME $S3_DATABASE_BACKUP_LOCATION",quiet:false)
                            }
                        }
                }
            }

            stage('Deploy')
            {
                steps
                {
                    script
                        {   

                            echo 'Deployment should be here.'
                        }
                }
            }            

            stage('Tag the deployed commmit')
            {
                steps
                {
                    script
                        {   
                            bat 'git tag $COMPLETION_TAG_NAME $GIT_COMMIT && git push origin --tags'
                        }
                }
            }
    }     	
    post
    {
        failure
        {
            //Rollback database
            script
            {
                if(HAS_DB_CHANGED)
                {
                    runPowershell("rollbackDatabase $REFERENCE_DATABASE_BACKUP_NAME $DATABASE_NAME",quiet:false)                
                }                
            }
        }              
    }
}

