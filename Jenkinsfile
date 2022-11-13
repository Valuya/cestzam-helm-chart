pipeline {
    agent {
        label 'cluster-manager'
    }
    parameters {
        booleanParam(name: 'PUSH', defaultValue: true, description: 'Push chart on repo')
        string(name: 'API_URI', defaultValue: 'https://nexus.valuya.com/service/rest/v1/components?repository=helm', description: 'Alternative deployment repo')
        credentials(name: 'API_CREDENTIAL',description: 'Repo credentials',credentialType: "Username with password", required: false,
         defaultValue: 'jenkins-jenkins-nexus-credentials')
        string(name: 'REPO_CONFIG_FILE', defaultValue: 'jenkins-helm-repository-config', description: 'Repo config file to use')
        string(name: 'GPG_KEY_CREDENTIAL_ID', defaultValue: 'jenkins-jenkins-charlyghislain-maven-deploy-gpg-key',
         description: 'Credential containing the private gpg key (pem)')
        string(name: 'GPG_KEY_FINGERPRINT', defaultValue: '508608F6CF097B4746CB291B5B72FDC1FF81F9ED',
         description: 'The fingerprint of this key to add to trust root')
        string(name: 'GPG_KEYNAME', defaultValue: 'jenkins-ossrh-deploy', description: 'The key name')
    }
    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
        stage ('Push') {
            when { anyOf {
              expression { return params.PUSH == true }
            } }
            steps {
                container('manager') {
                  withCredentials([file(credentialsId: params.GPG_KEY_CREDENTIAL_ID, variable: 'GPGKEY')]) {
                  withCredentials([usernamePassword(credentialsId: params.API_CREDENTIAL, passwordVariable: 'PASSWORD', usernameVariable: 'USER')]) {
                    configFileProvider([configFile(fileId: params.REPO_CONFIG_FILE, replaceTokens: true, variable: 'REPOCONFFILE')]) {
                      sh "gpg --batch --allow-secret-key-import --import $GPGKEY"
                      sh "echo \"${params.GPG_KEY_FINGERPRINT}:6:\" | gpg --batch --import-ownertrust"
                      sh "gpg --batch --export-secret-keys  -o /tmp/helmring.gpg"
                      sh '''
                       helm repo update --repository-cache .repo/ --repository-config "${REPOCONFFILE}"
                       OUT="$(helm package .  -u --repository-cache .repo/ --repository-config "${REPOCONFFILE}" --key "${GPG_KEYNAME}" --keyring  /tmp/helmring.gpg --sign)"
                       OUT_PATH="$(echo "$OUT" | cut -d':' -f2 | xargs)"
                       echo "Packaged $CHART_PATH to $OUT_PATH"

                       curl --fail -vF file=@"$OUT_PATH" -u"${USER}:${PASSWORD}" "${API_URI}" \
                        && echo "Pushed $(basename ${OUT_PATH}) to ${API_URI}" || (echo "!!! FAILED TO PUSH ($?)" && exit 1)
                      '''
                    }
                  }}
                }
            }
             post {
                failure {
                  echo "Push failed"
                }
             }
        }
    }
}
