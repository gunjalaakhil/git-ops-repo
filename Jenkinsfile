pipeline {

    agent any

    environment {

        APP_NAME = "mychart"
        ARGOCD_URL = "https://34.228.63.131:8080"

    }
    parameters {

    string(
        name: 'PR_NUMBER',
        defaultValue: '',
        description: 'GitHub Pull Request Number'
    )

}

    stages {

        stage('Checkout') {

            steps {

                checkout scm

            }

        }

        stage('Helm Lint') {

            steps {

                sh '''
                helm lint .
                '''

            }

        }

        stage('Helm Template Validation') {

            steps {

                sh '''
                helm template \
                mychart \
                . \
                -f values.yaml > rendered.yaml
                '''

            }

        }

        stage('Helm Diff Review') {

            steps {

                sh '''
                helm diff upgrade \
                mychart \
                . \
                -f values.yaml \
                --allow-unreleased || true
                '''

            }

        }

        stage('Approval') {

            steps {

                input(
                    message: 'Review Helm Diff and Approve Deployment',
                    ok: 'Merge & Deploy'
                )

            }

        }

 stage('Merge PR') {

    steps {

        withCredentials([
            string(
                credentialsId: 'github-token',
                variable: 'GITHUB_TOKEN'
            )
        ]) {

            sh '''

            echo "$GITHUB_TOKEN" | gh auth login --with-token

            gh pr merge ${PR_NUMBER} \
            --merge \
            --delete-branch

            '''

        }

    }

}
        stage('Capture Deployment Commit') {

            steps {

                script {

                    env.DEPLOY_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Deploy Commit = ${env.DEPLOY_COMMIT}"

                }

            }

        }

        stage('ArgoCD Login') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'argocd-creds',
                        usernameVariable: 'ARGO_USER',
                        passwordVariable: 'ARGO_PASS'
                    )
                ]) {

                    sh '''

                    argocd login \
                    $ARGOCD_URL \
                    --username $ARGO_USER \
                    --password $ARGO_PASS \
                    --insecure

                    '''

                }

            }

        }

        /*
        Auto-sync already enabled.
        We only wait for sync.
        */

        stage('Wait For Sync') {

            steps {

                sh '''

                argocd app wait \
                mychart \
                --sync \
                --timeout 300

                '''

            }

        }

        stage('Health Check') {

            steps {

                script {

                    try {

                        sh '''

                        argocd app wait \
                        mychart \
                        --health \
                        --sync \
                        --timeout 300

                        '''

                        sh '''

                        argocd app get mychart

                        '''

                        env.DEPLOY_STATUS = "SUCCESS"

                    }

                    catch(Exception ex) {

                        env.DEPLOY_STATUS = "FAILED"

                    }

                }

            }

        }

        stage('Deployment Decision') {

            when {

                expression {
                    env.DEPLOY_STATUS == "FAILED"
                }

            }

            steps {

                script {

                    env.DEPLOY_ACTION = input(

                        id: 'DeployDecision',

                        message: 'Deployment unhealthy. Choose Action.',

                        parameters: [

                            choice(
                                name: 'ACTION',

                                choices: [
                                    'RETRY',
                                    'ROLLBACK',
                                    'ABORT'
                                ].join('\n'),

                                description: 'Select Action'

                            )
                        ]
                    )
                }

            }

        }

        stage('Retry Health Check') {

            when {

                expression {
                    env.DEPLOY_ACTION == "RETRY"
                }

            }

            steps {

                sh '''

                argocd app wait \
                mychart \
                --health \
                --sync \
                --timeout 300

                '''

                sh '''
                argocd app get mychart
                '''

            }

        }

stage('Rollback') {

    when {
        expression {
            env.DEPLOY_ACTION == 'ROLLBACK'
        }
    }

    steps {

        withCredentials([
            string(
                credentialsId: 'github-token',
                variable: 'GITHUB_TOKEN'
            )
        ]) {

            sh """
                cd gitops-repo

                git config user.email "gakhil0071@gmail.com"
                git config user.name "gunjalakhil"

                git revert ${DEPLOY_COMMIT} --no-edit

                git push https://github.com/gunjalaakhil/git-ops-repo.git main
            """
        }
    }
}


        stage('Rollback Validation') {

            when {

                expression {
                    env.DEPLOY_ACTION == "ROLLBACK"
                }

            }

            steps {

                sh '''

                argocd app wait \
                mychart \
                --health \
                --sync \
                --timeout 300

                '''

            }

        }

        stage('Abort') {

            when {

                expression {
                    env.DEPLOY_ACTION == "ABORT"
                }

            }

            steps {

                error(
                    "Deployment Aborted By User"
                )

            }

        }

    }

    post {

        success {

            echo '''
==========================================
DEPLOYMENT SUCCESSFUL
==========================================
'''

        }

        failure {

            echo '''
==========================================
DEPLOYMENT FAILED
==========================================
'''

        }

        aborted {

            echo '''
==========================================
DEPLOYMENT ABORTED
==========================================
'''

        }

    }

}
