def runPowershell(cmd,quiet = true) {
       def script = "powershell -ExecutionPolicy ByPass -command "+cmd;
       if(!quiet)
        script = "@echo off && " + script;

       return bat(returnStdout:true , script: script).trim()
}

def backupDatabase()
{
    def backupCommand = "backupDatabase -databaseName $PROJECT -s3_object_arn_to_backup_to $DATABASE_BACKUP_LOCATION -databaseServer $DATABASE_SERVER -databaseUsername $DATABASE_MASTER_USERNAME -databasePassword $DATABASE_MASTER_PASSWORD"
    run(backupCommand,quiet:false)
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
                            HAS_DB_CHANGE_SCRIPTS = false                            
                            IS_APPLICATION_OFFLINE = false                          
                            DATABASE_BACKUP_LOCATION="$S3_BACKUPS_BUCKET/$PROJECT/" + new Date().format("dd.MM.yyyy@hh.mm.ss") + ".bak"
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

                            def changeScripts = runPowershell("getDBChangeScripts")
                            
                            if(changeScripts?.trim())
                            {
                                echo "Backing up the database"
                                backupDatabase();
                                echo "Backed up the database"
                            }

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
                if(ROLLBACK_DATABASE)
                {
                    rollbackDatabase();                 
                }                
            }
        }              
    }
}

