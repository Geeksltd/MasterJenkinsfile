def runPowershell(cmd,quiet=true) { 
        
       //cmd = '$ErrorActionPreference="Stop"; ' + cmd;
       def script ="powershell -ExecutionPolicy ByPass -command \""+cmd+"\"";
       
       if(quiet == true)
       {
           echo "running in quiet mode." + script
           script = "@echo off && " + script;   
       }

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
        IMAGE_BUILD_VERSION_TAG = "${GIT_COMMIT}v_${BUILD_NUMBER}"
		IMAGE_BUILD_VERSION = "${CONTIANER_REPOSITORY_URL}:${IMAGE_BUILD_VERSION_TAG}" 
		IMAGE_LATEST_VERSION = "${CONTIANER_REPOSITORY_URL}:latest" 		
		PROJECT_REPOSITORY_PASSWORD="$PROJECT_REPOSITORY_PASSWORD"
		PROJECT_REPOSITORY_USERNAME="$PROJECT_REPOSITORY_USERNAME"
		PROJECT_REPOSITORY_URL = "$PROJECT_REPOSITORY_URL"
        PROJECT_REPOSITORY_BRANCH = "$BRANCH"
        AWS_REGION="$REGION"                        
        ErrorActionPreference ="Stop"  
        DATABASE_NAME="$DATABASE_NAME"    
        S3_BACKUPS_BUCKET="$S3_BACKUPS_BUCKET"        
        PROJECT="$PROJECT"        
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
                            APPLY_CHANGE_SCRIPTS = false;
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
			stage("Load database changes") 
            { 
                steps { 
                    script 
                        {
                            CHANGE_SCRIPTS = runPowershell("getDBChangeScripts -lastSuccessfulDeploymentCommit $GIT_PREVIOUS_SUCCESSFUL_COMMIT")
                        }
                    }           
            }
            stage("Confirm database change scripts") {
                when { expression { CHANGE_SCRIPTS } }
                steps {
                    script {
                            APPLY_CHANGE_SCRIPTS = input message: "Detected [$CHANGE_SCRIPTS] scripts in this release.", 
                                                         ok: 'Continue',
                                                         parameters: [booleanParam(defaultValue: true, description: 'Apply the database changes?', name: 'applyChanges')]                    
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
                when { expression { CHANGE_SCRIPTS && APPLY_CHANGE_SCRIPTS } }    
                steps
                {
                    script
                        {   
                            
                           if(runPowershell("getDBChangeScripts")) 
                            {
                                HAS_DB_CHANGED = true;
                                runPowershell("applyDatabaseChanges $DATABASE_NAME $REFERENCE_DATABASE_NAME $S3_DATABASE_BACKUP_LOCATION",false)
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

                            newTaskDefinitionArn = runPowershell("registerNewTaskRevision -newImage $IMAGE_BUILD_VERSION -taskName $TASK_DEFINITION_NAME -region $REGION")

                            runPowershell("updateService -clusterName $CLUSTER_NAME -serviceName $SERVICE_NAME -newTaskDefinitionArn $newTaskDefinitionArn -region $REGION",false);
                            
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
                    // There is an option to restore the backup taken from S3. For now it is faster to rename the reference database.
                    runPowershell("rollbackDatabase $DATABASE_NAME $REFERENCE_DATABASE_NAME",false)                                                  
                }                
            }
        }              
    }
}

