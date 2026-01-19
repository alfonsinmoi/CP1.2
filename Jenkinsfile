pipeline {
    agent any

    stages {
        stage('Get Code') {
            steps {
                git branch: 'feature_fix_coverage', url: 'https://github.com/alfonsinmoi/CP1.2.git'
                echo "WORKSPACE: ${WORKSPACE}"
                sh 'ls -la'
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    export PYTHONPATH=${WORKSPACE}
                    python3 -m coverage run --branch --source=app --omit=app/__init__.py,app/api.py -m pytest --junitxml=result-unit.xml test/unit/
                    python3 -m coverage xml
                '''
            }
            post {
                always {
                    junit 'result-unit.xml'
                }
            }
        }

        stage('Coverage') {
            steps {
                sh '''
                    python3 -m coverage report
                '''
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    cobertura coberturaReportFile: 'coverage.xml',
                        conditionalCoverageTargets: '90,80,80',
                        lineCoverageTargets: '95,85,85',
                        failUnstable: false
                }
            }
        }

        stage('Static') {
            steps {
                sh '''
                    python3 -m flake8 --format=pylint --exit-zero app > flake8.out
                '''
                recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')],
                    qualityGates: [
                        [threshold: 8, type: 'TOTAL', unstable: true],
                        [threshold: 10, type: 'TOTAL', unstable: false]
                    ]
            }
        }

        stage('Security') {
            steps {
                sh '''
                    python3 -m bandit --exit-zero -r app -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                '''
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')],
                    qualityGates: [
                        [threshold: 2, type: 'TOTAL', unstable: true],
                        [threshold: 4, type: 'TOTAL', unstable: false]
                    ]
            }
        }

        stage('Rest Tests') {
            steps {
                sh '''
                    export FLASK_APP=${WORKSPACE}/app/api.py
                    python3 -m flask run --host=0.0.0.0 --port=5050 &
                    FLASK_PID=$!

                    java -jar /Users/macminimoi/wiremock/wiremock-standalone-3.10.0.jar --port 9090 --root-dir ${WORKSPACE}/test/wiremock &
                    WIREMOCK_PID=$!

                    echo "Esperando a que Flask este listo..."
                    for i in $(seq 1 30); do
                        if curl -s http://localhost:5050/ > /dev/null 2>&1; then
                            echo "Flask listo despues de $i intentos"
                            break
                        fi
                        sleep 1
                    done

                    echo "Esperando a que Wiremock este listo..."
                    for i in $(seq 1 30); do
                        if curl -s http://localhost:9090/__admin/ > /dev/null 2>&1; then
                            echo "Wiremock listo despues de $i intentos"
                            break
                        fi
                        sleep 1
                    done

                    curl -s http://localhost:5050/ || (echo "ERROR: Flask no responde" && exit 1)
                    curl -s http://localhost:9090/__admin/ || (echo "ERROR: Wiremock no responde" && exit 1)

                    export PYTHONPATH=${WORKSPACE}
                    python3 -m pytest --junitxml=result-rest.xml test/rest/

                    kill $FLASK_PID || true
                    kill $WIREMOCK_PID || true
                '''
            }
            post {
                always {
                    junit 'result-rest.xml'
                }
            }
        }

        stage('Performance') {
            steps {
                sh '''
                    export FLASK_APP=${WORKSPACE}/app/api.py
                    python3 -m flask run --host=0.0.0.0 --port=5050 &
                    FLASK_PID=$!

                    echo "Esperando a que Flask este listo..."
                    for i in $(seq 1 30); do
                        if curl -s http://localhost:5050/ > /dev/null 2>&1; then
                            echo "Flask listo despues de $i intentos"
                            break
                        fi
                        sleep 1
                    done

                    /Users/macminimoi/apache-jmeter-5.6.3/bin/jmeter -n -t test/jmeter/flask.jmx -l result-jmeter.jtl

                    kill $FLASK_PID || true
                '''
                perfReport sourceDataFiles: 'result-jmeter.jtl'
            }
        }
    }

    post {
        always {
            sh '''
                pkill -f "flask" || true
                pkill -f "wiremock" || true
            '''
            cleanWs()
        }
    }
}
