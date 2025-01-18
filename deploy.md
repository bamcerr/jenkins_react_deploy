pipeline {
    agent any

    environment {
        NGINX_HOST = "nginx"            // Nginx 서버 호스트명 또는 IP
        SSH_KEY = "/var/jenkins_home/.ssh/jenkins_rsa"     // Jenkins에서 사용할 SSH 키 경로
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM', 
                    branches: [[name: '*/nginx-deploy']], 
                    userRemoteConfigs: [
                        [
                            url: 'https://github.com/bamcerr/react-nomflix.git',
                            credentialsId: 'github-crd'  // Jenkins에 등록된 자격 증명 ID
                        ]
                    ]
                ])
            }
        }
        
        stage('Build') {
            steps {
                script {
                 // npm ci로 의존성 설치 (CI 환경에 더 적합)
                    sh 'npm ci'
                    // React 앱 빌드
                    sh 'npm run build'
                }
            }
        }
        stage('Archive Build') {
            steps {
                // 빌드된 파일 압축
                sh '''
                    tar -czf nomflix.tar.gz -C build .
                '''
            }
        }

        stage('Deploy to Nginx Server') {  // Nginx 서버에 배포
            steps {
                // script {
                //     // SSH로 Nginx 서버에 접속하여 이미지를 풀하고 컨테이너를 재시작합니다.
                //     sh '''
                //         ssh -i /var/jenkins_home/.ssh/jenkins_rsa -o StrictHostKeyChecking=no root@nginx -p 22 "nginx -v"
                //     '''
                // }
               
                sshagent(credentials: ['nginx-crd']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no root@nginx "
                            if \\$(find /var/www/nomflix/build -mindepth 1 -print -quit | grep -q .); then 
                                tar -czf /var/www/nomflix/backup/nomflix_\\$(date +%Y%m%d%H%M%S).tar.gz /var/www/nomflix/build/*;
                            else
                                echo "No files to archive in /var/www/nomflix/build";
                            fi
                        "
                        
                        scp nomflix.tar.gz root@nginx:/var/www/nomflix/temp/
                        
                        ssh root@nginx "
                            tar -xzf /var/www/nomflix/temp/nomflix.tar.gz -C /var/www/nomflix/build && \
                            rm /var/www/nomflix/temp/nomflix.tar.gz
                        "
                    '''
                    
                }
            }
        }
    }

    post {
        always {
            // 빌드 후 정리 작업. 워크스페이스를 청소합니다.
            cleanWs()
        }
    }
}
