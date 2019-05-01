def GetPowershellResult(cmd) { 
        
       //cmd = '$ErrorActionPreference="Stop"; ' + cmd;
       def script ="powershell -ExecutionPolicy ByPass -command \""+cmd+"\"";
       
       echo "running in quiet mode." + script
       script = "@echo off && " + script;   
       
       return bat(returnStdout:true , script: script).trim()
}

def RunPowershell(cmd)
{
    def script ="powershell -ExecutionPolicy ByPass -command \""+cmd+"\"";
       
    bat script
}

def GetEnvironmentVariablesHashTable()
{
    def settings = System.getenv("RUNTIME_ENVIRONMENT_VARIABLES");

    if(!settings) return "";

    return "{ " + new groovy.json.JsonSlurper().parseText(settings).collect { k,v -> "'$k'='$v'" }.join(' ; ') + " }"
}

import com.cloudbees.plugins.credentials.impl.*;
import com.cloudbees.plugins.credentials.*;
import com.cloudbees.plugins.credentials.domains.*;

pipeline 
{
    parameters 
    {     
        booleanParam(name: "rebuildDockerImage" , defaultValue : false)
    }
    environment 
    {   
        REPOSITORY_CREDENTIALS_ID = "REPOSITORY_CREDENTIALS"	
        IMAGE_BUILD_VERSION_TAG = "${CONTIANER_IMAGE_TAG_PREFIX}_${GIT_COMMIT}_v_${BUILD_NUMBER}"        
		IMAGE_BUILD_VERSION = "${CONTIANER_REPOSITORY_URL}:${IMAGE_BUILD_VERSION_TAG}".toLowerCase()
		IMAGE_LATEST_VERSION = "${CONTIANER_REPOSITORY_URL}:${CONTIANER_IMAGE_TAG_PREFIX}_latest".toLowerCase()
		PROJECT_REPOSITORY_PASSWORD="$PROJECT_REPOSITORY_PASSWORD"
		PROJECT_REPOSITORY_USERNAME="$PROJECT_REPOSITORY_USERNAME"
		PROJECT_REPOSITORY_URL = "$PROJECT_REPOSITORY_URL"
        PROJECT_REPOSITORY_BRANCH = "$BRANCH"
        AWS_REGION="$REGION"                        
        ErrorActionPreference ="Stop"  
        DATABASE_NAME="$DATABASE_NAME"            
        PROJECT="$PROJECT"
        TASK_DEFINITION_NAME="$PROJECT_TASK_FAMILY_NAME"
        SERVICE_NAME="$PROJECT_ECS_SERVICE_NAME"
        CLUSTER_NAME="$ECS_CLUSTER_NAME"        
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
                            DATE_TAG=new Date().format('dd.MM.yyyy@hh.mm.ss');
                            REFERENCE_DATABASE_NAME = "$DATABASE_NAME" + "_" + DATE_TAG;                                                        
                            APPLY_CHANGE_SCRIPTS = false;
                            PREVIOUS_SUCCESSFUL_DEPLOYMENT_COMMIT= System.getenv("GIT_PREVIOUS_SUCCESSFUL_COMMIT") ?: ""
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
                            CHANGE_SCRIPTS = GetPowershellResult("getDBChangeScripts $PREVIOUS_SUCCESSFUL_DEPLOYMENT_COMMIT")
                        }
                    }           
            }
            stage("Confirm database change scripts") {
                when { expression { CHANGE_SCRIPTS } }
                steps {
                    script 
                    {
                        GIT_PREVIOUS_SUCCESSFUL_COMMIT_DATE= GetPowershellResult("getLastSuccessfulDeploymentCommitDate $PREVIOUS_SUCCESSFUL_DEPLOYMENT_COMMIT")                        
                        APPLY_CHANGE_SCRIPTS = input message: "Detected [$CHANGE_SCRIPTS] scripts in this release since $PREVIOUS_SUCCESSFUL_DEPLOYMENT_COMMIT.", 
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
                            RunPowershell("docker build -t $IMAGE_BUILD_VERSION" + (!params.rebuildDockerImage ? "" : " --no-cache") + " -t $IMAGE_LATEST_VERSION .")
                        }
                }				
            }
			stage('Push the image') 
            {
                steps
                {
                    script
                        {
							RunPowershell("""Invoke-Expression -Command (Get-ECRLoginCommand -Region eu-west-1).Command 
                                                                                 docker push $IMAGE_BUILD_VERSION
                                                                                 docker push $IMAGE_LATEST_VERSION""")
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
                          HAS_DB_CHANGED = true;
                          RunPowershell("applyDatabaseChanges $DATABASE_NAME $REFERENCE_DATABASE_NAME $DATABASE_BACKUP_S3_BUCKET $GIT_PREVIOUS_SUCCESSFUL_COMMIT")                          
                        }
                }
            }
            stage('Deploy')
            {
                steps
                {
                    script
                        {   

                            newTaskDefinitionArn = GetPowershellResult("registerNewTaskRevision -newImage $IMAGE_BUILD_VERSION -taskName $TASK_DEFINITION_NAME -region $REGION -environmentVariables " + GetEnvironmentVariablesHashTable())

                            RunPowershell("updateService -clusterName $CLUSTER_NAME -serviceName $SERVICE_NAME -newTaskDefinitionArn $newTaskDefinitionArn -region $REGION");
                            
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
                    RunPowershell("rollbackDatabase $DATABASE_NAME $REFERENCE_DATABASE_NAME")                                                  
                }                
            }
        }              
    }
}

