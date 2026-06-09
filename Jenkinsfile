pipeline {
    agent any

    environment {
        SCAN_TARGET  = 'http://localhost:4000'
        SEMGREP      = '/home/mamadbah/projects/semgroup-venv/bin/semgrep'
        NODE_BIN     = '/home/mamadbah/.nvm/versions/node/v24.14.1/bin'
        NOTIFY_EMAIL = 'bahmamadoubobosewa@gmail.com'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Extraction du code source...'
                checkout scm
            }
        }

        stage('SCA - npm audit') {
            steps {
                sh '''
                    export PATH=$NODE_BIN:$PATH
                    echo "Analyse des dependances (SCA)..."
                    npm audit --json > sca-npm-audit.json || true
                    npm audit         > sca-npm-audit.txt  || true
                '''
                archiveArtifacts artifacts: 'sca-npm-audit.*', allowEmptyArchive: true
            }
        }

        stage('SAST - Semgrep') {
            steps {
                sh '''
                    echo "Analyse statique du code (SAST)..."
                    # Config epinglee (p/default) + metrics off : pas d'envoi de telemetrie a Semgrep
                    "$SEMGREP" scan --config "p/default" --metrics off \
                        --sarif-output semgrep-sast.sarif \
                        --json-output  semgrep-sast.json \
                        --text-output  semgrep-sast.txt || true
                '''
                archiveArtifacts artifacts: 'semgrep-sast.*', allowEmptyArchive: true
            }
        }

        stage('Demarrage Juice Shop') {
            steps {
                sh '''
                    docker rm -f juiceshop-test >/dev/null 2>&1 || true
                    # Image epinglee par digest (supply-chain) ; publiee sur loopback uniquement
                    # car l'appli est volontairement vulnerable -> joignable par ZAP, pas par le reseau
                    docker run -d --name juiceshop-test -p 127.0.0.1:4000:3000 \
                        bkimminich/juice-shop@sha256:25fd268112350ae9e0ddc7878371f9f12f5b0b546c7bf934d6599aa8e724418f

                    echo "Attente du demarrage de Juice Shop..."
                    for i in $(seq 1 40); do
                        if curl -sf http://localhost:4000 >/dev/null 2>&1; then
                            echo "Juice Shop est pret."
                            exit 0
                        fi
                        sleep 3
                    done
                    echo "ATTENTION : Juice Shop n'a pas repondu a temps."
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    echo "Scan dynamique (DAST)..."
                    # Le workspace doit etre accessible en ecriture par l'utilisateur du conteneur ZAP (uid 1000)
                    chmod 777 "$WORKSPACE" || true
                    # --network host : ZAP atteint Juice Shop via localhost:4000 sur la machine hote
                    docker run --rm --network host \
                        -v "$WORKSPACE":/zap/wrk/:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py -t "$SCAN_TARGET" \
                        -r zap-dast-report.html \
                        -J zap-dast-report.json || true
                '''
                archiveArtifacts artifacts: 'zap-dast-report.*', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "Arret de Juice Shop..."
            sh 'docker rm -f juiceshop-test >/dev/null 2>&1 || true'
        }

        success {
            emailext(
                to: "${env.NOTIFY_EMAIL}",
                subject: "[Jenkins] Succes - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2 style="color:#2e7d32">Pipeline securite : SUCCES</h2>
                    <p><b>Job :</b> ${env.JOB_NAME} #${env.BUILD_NUMBER}</p>
                    <p>Les analyses <b>SAST (Semgrep)</b>, <b>SCA (npm audit)</b> et
                       <b>DAST (OWASP ZAP)</b> se sont executees.</p>
                    <p>Rapports lisibles en pieces jointes ; tous les rapports
                       (JSON/SARIF/HTML) sont archives sur le build.</p>
                    <p><a href="${env.BUILD_URL}">Ouvrir le build sur Jenkins</a></p>
                """,
                attachmentsPattern: 'semgrep-sast.txt, sca-npm-audit.txt, zap-dast-report.html'
            )
        }

        failure {
            emailext(
                to: "${env.NOTIFY_EMAIL}",
                subject: "[Jenkins] Echec - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2 style="color:#c62828">Pipeline securite : ECHEC</h2>
                    <p><b>Job :</b> ${env.JOB_NAME} #${env.BUILD_NUMBER}</p>
                    <p>Le pipeline a echoue. Consultez le journal (joint) ou la console.</p>
                    <p><a href="${env.BUILD_URL}console">Voir les logs Jenkins</a></p>
                """,
                attachLog: true
            )
        }
    }
}
