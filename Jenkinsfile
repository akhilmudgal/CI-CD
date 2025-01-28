pipeline {
    agent any

    environment {
	APPLICATION = 'proxy-next'
	ENV_NAME = 'prod-proxy-next'
        VROLE = credentials('vault_role')
        VSID  = credentials('vault_sid')
        REPO_URL = 'git@github.com:DevnagriAI/dota-proxy.git'
        BRANCH = 'next'
        GCP_PROJECT = 'striking-canyon-392409'
        GCP_ZONE = 'asia-south1-a'
        INSTANCE_NAME = 'preprod-proxy-next'
        USER = 'ubuntu' 	
	BUILD_NUMBER= "${env.BUILD_NUMBER}"
        IG_NAME = 'proxy-next-autoscalling-ig'
        IMAGE_NAME = "${INSTANCE_NAME}-prod-image-v${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${env.BRANCH}",credentialsId: 'github', url: "${env.REPO_URL}"
            }
        }


	 stage('SonarQube Analysis') {
            steps {
                   script {
                       scannerHome = tool 'SonarQubeScanner'
        }
        withSonarQubeEnv('SonarQube') {
          sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${APPLICATION}"
                }
            }
        }
        
        stage('Start PreProd Instance') {
            steps {
                script {
                    sh '''
                        gcloud compute instances start ${INSTANCE_NAME} --zone=${GCP_ZONE}
                    '''
                }
            }
        }

        stage('Rsync Code to Instance') {
            steps {
                script {
                    sh '''
                        INSTANCE_IP=$(gcloud compute instances describe ${INSTANCE_NAME} --zone=${GCP_ZONE} --format='get(networkInterfaces[0].networkIP)')
			/var/lib/jenkins/scripts/vault.sh
			sleep 10
                        rsync -e "ssh -o StrictHostKeyChecking=no" -avz /var/lib/jenkins/vault/proxy-next.env ${USER}@${INSTANCE_IP}:/home/${USER}/proxy.devnagri.com/.env
                        rsync -e "ssh -o StrictHostKeyChecking=no" --exclude='template_properties.json' --exclude='.git/' -avz . ${USER}@${INSTANCE_IP}:/home/${USER}/proxy.devnagri.com/
                        ssh -o StrictHostKeyChecking=no ${USER}@${INSTANCE_IP} 'cd proxy.devnagri.com;npm i'
			ssh -o StrictHostKeyChecking=no ${USER}@${INSTANCE_IP} 'cd proxy.devnagri.com;npm run lint:fix'
                        ssh -o StrictHostKeyChecking=no ${USER}@${INSTANCE_IP} 'cd proxy.devnagri.com;npm run prod:build'
                        ssh -o StrictHostKeyChecking=no ${USER}@${INSTANCE_IP} 'cd proxy.devnagri.com;pm2 restart all'

                    '''
                }
            }
        }

        stage('Stop GCP Instance') {
            steps {
                script {
                    sh '''
                        gcloud compute instances stop ${INSTANCE_NAME} --zone=${GCP_ZONE}
                    '''
                }
            }
        }
        
        stage('Create Image of Instance') {
            steps {
                script {
                    sh '''
			gcloud compute instance-groups managed set-autoscaling ${IG_NAME} --min-num-replicas=2 --max-num-replicas=3 --target-cpu-utilization=0.7  --zone=asia-south1-a
                        gcloud compute images create ${IMAGE_NAME} --source-disk=${INSTANCE_NAME} --source-disk-zone=${GCP_ZONE} --family=${INSTANCE_NAME}-family
                    '''
                }
            }
        }

      stage('Update Instance Group and Template') {
            steps {
                script {
                    sh '''
                            INSTANCE_TEMPLATE=$(gcloud compute instance-groups managed describe ${IG_NAME} --zone=${GCP_ZONE} --format="get(instanceTemplate)")
                            
			    TEMPLATE_NAME=$(basename ${INSTANCE_TEMPLATE})
		            gcloud compute instance-templates describe ${TEMPLATE_NAME} --format="json" > template_properties.json
                            MACHINE_TYPE=$(jq -r '.properties.machineType' template_properties.json)
                            NETWORK=$(jq -r '.properties.networkInterfaces[0].network' template_properties.json)
                            TAGS=$(jq -r '.properties.tags.items[]' template_properties.json | tr '\n' ',' | sed 's/,$//')
                            SERVICE_ACCOUNT=$(jq -r '.properties.serviceAccounts[0].email' template_properties.json)
                            SCOPES=$(jq -r '.properties.serviceAccounts[0].scopes[]' template_properties.json | tr '\n' ',' | sed 's/,$//')
			    DISK_SIZE=$(jq -r '.properties.disks[0].initializeParams.diskSizeGb' template_properties.json)
			    DISK_TYPE=$(jq -r '.properties.disks[0].initializeParams.diskType' template_properties.json)
			    LABELS=$(jq -r '.properties.labels | to_entries[] | "--labels \\(.key)=\\(.value)"' template_properties.json | tr '\n' ',' | sed 's/,$//')

                            gcloud compute instance-templates create ${APPLICATION}-template-v${BUILD_NUMBER} --machine-type=${MACHINE_TYPE} --network=${NETWORK} --tags=${TAGS} --service-account=${SERVICE_ACCOUNT} --scopes=${SCOPES} --boot-disk-device-name=${APPLICATION}-template-v${BUILD_NUMBER}  --boot-disk-auto-delete --boot-disk-size=${DISK_SIZE} --boot-disk-type=${DISK_TYPE} --image=${IMAGE_NAME} --shielded-integrity-monitoring --shielded-secure-boot --shielded-vtpm --labels=environment=prod,name=${APPLICATION}
			    gcloud compute instance-groups managed update ${IG_NAME} --update-policy-type=opportunistic
			    gcloud compute instance-groups managed set-instance-template proxy-next-autoscalling-ig --zone=asia-south1-a --template=${APPLICATION}-template-v${BUILD_NUMBER}
			    sleep 60
			    gcloud compute instance-groups managed rolling-action start-update ${IG_NAME} --zone=${GCP_ZONE} --max-surge=1 --version=template=${APPLICATION}-template-v${BUILD_NUMBER}
			    sleep 120
			    gcloud compute instance-groups managed set-autoscaling ${IG_NAME} --min-num-replicas=1 --max-num-replicas=3 --target-cpu-utilization=0.7  --zone=asia-south1-a  
                    # Clean up the temporary files
                    rm template_properties.json 
		    
                    '''
                }
            }
        }


    }
    
    post {
        always {
            script {
                sh '''
                    gcloud compute instances stop ${INSTANCE_NAME} --zone=${GCP_ZONE}
		    echo "All good for now"
                '''
            }
        }
    }
}

