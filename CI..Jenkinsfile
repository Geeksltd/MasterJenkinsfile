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
    }
    agent any
    stages
	{    
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
          
    }     	
}

