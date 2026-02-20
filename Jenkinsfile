pipeline{
    agent any
    
    options {skipDefaultCheckout()}
    
    stages{
        stage('Get Code'){
            steps{
                git url: 'https://github.com/jorgesaintremy/tod-list-aws.git', branch: 'master'
                echo WORKSPACE
                sh 'ls -lart'
                sh 'whoami'
                sh 'hostname'
                stash name: 'code', includes: '**'
                deleteDir()
            }
        }
        stage('Deploy'){
            agent{label 'ubuntu_aws'}
            steps{
                unstash name:'code'
                echo WORKSPACE
                sh '''
                    ls -lart
                    whoami
                    hostname
                    sam build
                    sam validate --region us-east-1
                    sam deploy --config-file samconfig.toml --config-env production
                '''
                deleteDir()
                }
            }
        stage('Rest Test'){
            agent{label 'ubuntu_aws'}
            environment{
                BASE_URL = sh "aws cloudformation describe-stacks --stack-name todo-list-aws-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text"
            }
            steps{
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                    unstash name:'code'
                    echo WORKSPACE
                    sh '''
                        ls -lart
                        whoami
                        hostname
                        export BASE_URL=$(aws cloudformation describe-stacks --stack-name todo-list-aws-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text)
                        echo $BASE_URL
                        export PYTHONPATH=${WORKSPACE}
                        python3 -m pytest -v -m read --junitxml=result-integration.xml test/integration/todoApiTest.py
                        cat result-integration.xml                       
                    '''
                    stash name: 'integration-res', includes: 'result-integration.xml'
                    junit 'result*.xml'
                    deleteDir()
                }
            }
        }
    }
}
