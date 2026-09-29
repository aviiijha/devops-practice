pipeline {
    agent any

    environment {
        SITE            = 'devops-practice'
        PANTHEON_REMOTE = 'ssh://codeserver.dev.d7eee6a9-45ab-4611-8841-ef73cb93c2f8@codeserver.dev.d7eee6a9-45ab-4611-8841-ef73cb93c2f8.drush.in:2222/~/repository.git'
        TERMINUS_TOKEN  = credentials('pantheon-machine-token')
        // Jenkins (brew service) doesn't load your shell profile, so add Homebrew's bin
        PATH = "/opt/homebrew/bin:/usr/local/bin:${env.PATH}"
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate & Test') {
            steps {
                sh 'composer validate --no-check-publish'
                sh 'composer install --no-interaction --prefer-dist --no-progress'
                // Syntax-check any custom PHP code (none yet on a fresh site)
                sh '''
                    if [ -d web/modules/custom ]; then
                        find web/modules/custom -name "*.php" -o -name "*.module" -o -name "*.inc" | xargs -r -n1 php -l
                    fi
                '''
            }
        }

        stage('Deploy to Pantheon Dev') {
            steps {
                sshagent(['pantheon-ssh-key']) {
                    sh '''
                        git remote add pantheon "$PANTHEON_REMOTE" 2>/dev/null || git remote set-url pantheon "$PANTHEON_REMOTE"
                        export GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=accept-new"
                        git push pantheon HEAD:master
                    '''
                }
            }
        }

        stage('Post-deploy on Dev') {
            options {
                timeout(time: 15, unit: 'MINUTES')
            }
            steps {
                sh '''
                    terminus auth:login --machine-token="$TERMINUS_TOKEN"
                    # Wait for Pantheon's Integrated Composer build + code sync to finish.
                    # (terminus workflow:wait can hang after the workflow finishes, so poll instead.)
                    sleep 15
                    for i in $(seq 1 60); do
                        STATUS=$(terminus workflow:info:status "$SITE" --field=status 2>/dev/null)
                        echo "Latest Pantheon workflow: $STATUS"
                        [ "$STATUS" = "succeeded" ] && break
                        [ "$STATUS" = "failed" ] && { echo "Pantheon workflow failed"; exit 1; }
                        sleep 10
                    done
                    terminus drush "$SITE.dev" -- updatedb -y
                    terminus drush "$SITE.dev" -- cache:rebuild
                '''
            }
        }

        stage('Deploy to Test') {
            steps {
                input message: 'Deploy to Pantheon TEST?'
                sh 'terminus env:deploy "$SITE.test" --updatedb --cc --note="Jenkins build #$BUILD_NUMBER"'
            }
        }

        stage('Deploy to Live') {
            steps {
                input message: 'Deploy to Pantheon LIVE?'
                sh 'terminus env:deploy "$SITE.live" --updatedb --cc --note="Jenkins build #$BUILD_NUMBER"'
            }
        }
    }

    post {
        success { echo "Deployed ${env.GIT_COMMIT}" }
        failure { echo 'Build failed — check the stage logs above' }
    }
}
