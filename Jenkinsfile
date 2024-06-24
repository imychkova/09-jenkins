pipeline {
    agent any

    stages {
        stage('Setup') {
            steps {
                sh '''
                export PATH=$PATH:/usr/bin
                '''
            }
        }

        stage('Run Apache2 server') {
            steps {
                sh '''
                docker run -d --network jenkins   --name my-apache2 -p 80 httpd:2.4

                '''
            }
        }


        stage('send requests') {
            steps {
                sh '''
                curl -s -o /dev/null -w "%{http_code}" http://my-apache2:80/ || echo 'works 200'
                curl -s -o /dev/null -w "%{http_code}" http://my-apache2:80/404  || echo 'works 404'
                curl -x PUT -s -o /dev/null -w "%{http_code}" http://my-apache2:80/404  || echo 'works 404'
                curl -x POST -s -o /dev/null -w "%{http_code}" http://my-apache2:80/404  || echo 'works 404'
                '''
            }
        }


        stage('Check Apache2 logs for errors') {
            steps {
                sh '''
                docker logs my-apache2 | grep -E "[45][0-9][0-9]"
                '''
            }
        }
    }

    post {
        always {
            sh '''
            docker rm -fv my-apache2 || true
            '''
        }
    }
}