pipeline{
    agent none
    
    options {skipDefaultCheckout()}
    
    stages{
        stage('Get Code'){
            agent{label 'ubuntu_aws_2'}
            steps{
                git url: 'https://github.com/jorgesaintremy/tod-list-aws.git', branch: 'develop'
                echo WORKSPACE
                sh 'rm samconfig.toml'
                sh 'ls -lart'
                sh 'whoami'
                sh 'hostname'
                stash name: 'code', includes: '**'
                deleteDir()
            }
        }
        stage('Static'){
            agent{label 'ubuntu_aws'}
            steps{
                catchError(buildResult: 'UNSTABLE', stageResult: 'SUCCESS'){
                unstash name: 'code'
                echo WORKSPACE
                sh '''
                    ls -lart
                    whoami
                    hostname
                    export PYTHONPATH=$WORKSPACE
                    python3 -m flake8 --format=pylint --exit-zero app>flake8.out
                '''
                recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], qualityGates : [[threshold: 8, type: 'TOTAL', unstable: true], [threshold: 10, type: 'TOTAL', unstable: false]]
                stash name: 'static-res', includes: 'flake8.out'
                deleteDir()
                }
            }
        }
        stage('Security Test'){
            agent{label 'ubuntu_aws'}
            steps{
                catchError(buildResult: 'UNSTABLE', stageResult: 'SUCCESS'){
                unstash name:'code'
                echo WORKSPACE
                sh '''
                    ls -lart
                    whoami
                    hostname
                    python3 -m bandit -r ./src/ -f custom -o bandit.out --msg-template "{abspath}:{line}:[{test_id}] {msg}"
                '''
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')], qualityGates: [[threshold: 2, type: 'TOTAL', unstable: true], [threshold: 4, type: 'TOTAL', unstable: false]]
                deleteDir()
                }
            }
        }
        stage('Deploy'){
            agent{label 'ubuntu_aws'}
            steps{
                unstash name:'code'
                echo WORKSPACE
                sh '''
                    wget https://raw.githubusercontent.com/jorgesaintremy/todo-list-aws-config/refs/heads/staging/samconfig.toml 
                    ls -lart
                    whoami
                    hostname
                    sam build
                    sam validate --region us-east-1
                    sam deploy --config-file samconfig.toml --config-env staging
                '''
                deleteDir()
                }
            }
        stage('Rest Test'){
            agent{label 'ubuntu_aws_2'}
            environment{
                BASE_URL = sh "aws cloudformation describe-stacks --stack-name todo-list-aws-staging --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text"
            }
            steps{
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                    unstash name:'code'
                    echo WORKSPACE
                    sh '''
                        ls -lart
                        whoami
                        hostname
                        export BASE_URL=$(aws cloudformation describe-stacks --stack-name todo-list-aws-staging --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text)
                        echo $BASE_URL
                        export PYTHONPATH=${WORKSPACE}
                        python3 -m pytest -v --junitxml=result-integration.xml test/integration/todoApiTest.py
                        cat result-integration.xml                       
                    '''
                    stash name: 'integration-res', includes: 'result-integration.xml'
                    junit 'result*.xml'
                    deleteDir()
                }
            }
        }
        stage('Promote'){
            agent{label 'ubuntu_aws'}
            steps{
                withCredentials([usernamePassword(
                    credentialsId: 'b8405cce-0abf-4558-a38a-fb4feda9710c',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'PASSWORD'
                )]){
                git url: 'https://github.com/jorgesaintremy/tod-list-aws.git', branch: 'develop'
                echo WORKSPACE
                sh '''
                    ls -lart
                    whoami
                    hostname
                    pwd
                    git fetch https://$PASSWORD@github.com/jorgesaintremy/tod-list-aws.git
                    git remote -v
                    git checkout master
                    git config --global merge.ours.driver true
                    git merge origin/develop
                    git log --all --graph --decorate --oneline
                   
                '''
                deleteDir()
                }
            }
        }
    }
}
