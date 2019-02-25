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
            stage('Clone sources') 
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
                            dir("DB.Scripts/$BUILD_NUMBER/")        
                            {                           
                                withCredentials(usernamePassword(credentialsId: "$DB_CREDENTIALS",usernameVariable: 'DATABASE_USER', passwordVariable: 'DATABASE_PASSWORD'))
                                    {
                                        if(fileExists("/"))
                                        {

                                        }

                                        try
                                        {
                                            powershell """                                            
                                                        Get-ChildItem -Filter *.sql |                                   
                                                        Foreach-Object {
                                                            Write-Host "Running \$file"         
                                                            \$file = \$_.FullName                                                           
                                                            Invoke-Sqlcmd -ServerInstance $DATABASE_SERVER -Database $DATABASE_NAME -Username $DATABASE_USER -Password $DATABASE_PASSWORD -InputFile  \$file
                                                            Write-Host "Ran \$file"
                                                        }                                               
                                            """
                                        }
                                        catch(Exception e)
                                        {
                                            ROLLBACK_DATABASE = true
                                            error("Failed to update the database.");
                                        }
                                    }
                            }
                        }
                }
            }

            stage('Update the cluster') 
            {
                steps
                {
                    script
                        {
                            powershell label: '', returnStatus: true, script: """
                            \$TASK_REVISION=(aws ecs register-task-definition \
                            --family $PROJECT_TASK_FAMILY_NAME \
                            --task-role-arn $TASK_ROLE_ARN \
                            --region $AWS_REGION \
                            --container-definitions '[{ ""name"" : ""$PROJECT_TASK_FAMILY_NAME"" , ""image"":""$IMAGE_BUILD_VERSION"" , ""memory"" : 500 , ""portMappings"" : [ { ""containerPort"" : 80 , ""hostPort"" : 0 , ""protocol"" : ""tcp"" } ] , ""essential"" : true  }]') | ConvertFrom-Json | select -expand taskDefinition | Select revision

                            aws ecs update-service \
                            --cluster $ECS_CLUSTER_NAME \
                            --service $PROJECT_ECS_SERVICE_NAME \
                            --region $AWS_REGION \
                            --task-definition "$PROJECT_TASK_FAMILY_NAME:\$(\$TASK_REVISION.revision)"
                            """   
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

